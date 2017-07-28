---
title: WebKit渲染流程
date: 2015-04-09 14:51:03
categories: WebKit
tags: WebKit
---

注意：代码并不完整，只是摘录。

## UI 驱动

直接绘制 content(PictureSet)。

``` cpp
// WebKit/gaia/frameworks/webkit/WebView.cpp

void WebView::onDraw(const sp<Canvas>& canvas) {
  int saveCount = canvas->save();
  drawContent(canvas, drawNativeRings);
  canvas->restoreToCount(saveCount);

  if (AUTO_REDRAW_HACK && mAutoRedraw)
    invalidate();

  mWebViewCore->signalRepaintDone();
}

void WebView::drawContent(const sp<Canvas>& canvas, bool drawRings) {
  drawCoreAndCursorRing(canvas, mBackgroundColor, mDrawCursorRing && drawRings);
}

void WebView::drawCoreAndCursorRing(const sp<Canvas>& canvas, int color, bool drawCursorRing) {
  if (mDrawHistory) {
    canvas->scale(mZoomManager->getScale(), mZoomManager->getScale());
    canvas->drawPicture(mHistoryPicture);
    return;
  }

  int saveCount = canvas->save();
  if (animateZoom)
    mZoomManager->animateZoom(canvas);
  else if (!canvas->isHardwareAccelerated())
    canvas->scale(mZoomManager->getScale(), mZoomManager->getScale());

  calcOurContentVisibleRectF(mVisibleContentRect);

  if (!canvas->isHardwareAccelerated()) {
    sp<DrawFilter> df = NULL;

    if (mZoomManager->isZoomAnimating() || UIAnimationsRunning)
      df = mZoomFilter;
    else if (animateScroll)
      df = mScrollFilter;

    canvas->setDrawFilter(df);
    // XXX: Revisit splitting content.  Right now it causes a
    // synchronization problem with layers.
    int content = nativeDraw(canvas, mVisibleContentRect, color, extras, false);
    canvas->setDrawFilter(NULL);
    if (!mBlockWebkitViewMessages && content != 0)
      mWebViewCore->sendMessage(EventHub::SPLIT_PICTURE_SET, content, 0);
  }

  canvas->restoreToCount(saveCount);
}

// gaia/nav/WebView.cpp

int nativeDraw(WebKit::WebViewProxy* proxy, SkCanvas* canvas,
               SkRect visibleRect, int color,
               int extras, bool split) {
  WebView* webView = reinterpret_cast<WebView*>(proxy->mNativeClass);
  webView->setVisibleRect(visibleRect);
  PictureSet* pictureSet = webView->draw(canvas, color, extras, split);
  return reinterpret_cast<int>(pictureSet);
}

PictureSet* draw(SkCanvas* canvas, SkColor bgColor, int extras, bool split) {
  // draw the content of the base layer first
  PictureSet* content = m_baseLayer->content();
  int sc = canvas->save(SkCanvas::kClip_SaveFlag);
  canvas->clipRect(SkRect::MakeLTRB(0, 0, content->width(),
                                    content->height()), SkRegion::kDifference_Op);
  canvas->drawColor(bgColor);
  canvas->restoreToCount(sc);
  if (content->draw(canvas))
    ret = split ? new PictureSet(*content) : 0;
  return ret;
}
```

## WebCore 驱动

使得 content(PictureSet) 被更新。

``` cpp
void FrameView::layout(bool allowSubtree) {
  // Now update the positions of all layers.
  beginDeferredRepaints();
  IntPoint cachedOffset;
  if (m_doFullRepaint)
    root->view()->repaint(); // FIXME: This isn't really right, since the RenderView doesn't fully encompass the visibleContentRect(). It just happens
                             // to work out most of the time, since first layouts and printing don't have you scrolled anywhere.

  layer->updateLayerPositions((m_doFullRepaint ? 0 : RenderLayer::CheckForRepaint)
                                  | RenderLayer::IsCompositingUpdateRoot
                                  | RenderLayer::UpdateCompositingLayers,
                              subtree ? 0 : &cachedOffset);
  endDeferredRepaints();

#if USE(ACCELERATED_COMPOSITING)
  updateCompositingLayers();
#endif

  m_layoutCount++;
  ASSERT(!root->needsLayout());
  updateCanBlitOnScrollRecursively();

  if (document->hasListenerType(Document::OVERFLOWCHANGED_LISTENER))
    updateOverflowStatus(layoutWidth() < contentsWidth(),
                         layoutHeight() < contentsHeight());

  if (!m_hasPendingPostLayoutTasks) {
    if (!m_inSynchronousPostLayout && !inSubframeLayoutWithFrameFlattening) {
      m_inSynchronousPostLayout = true;
      // Calls resumeScheduledEvents()
      performPostLayoutTasks();
      m_inSynchronousPostLayout = false;
    }

    if (!m_hasPendingPostLayoutTasks &&
        (needsLayout() || m_inSynchronousPostLayout || inSubframeLayoutWithFrameFlattening)) {
      // If we need layout or are already in a synchronous call to postLayoutTasks(),
      // defer widget updates and event dispatch until after we return. postLayoutTasks()
      // can make us need to update again, and we can get stuck in a nasty cycle unless
      // we call it through the timer here.
      m_hasPendingPostLayoutTasks = true;
      m_postLayoutTasksTimer.startOneShot(0);
      if (needsLayout()) {
        m_actionScheduler->pause();
        layout();
      }
    }
  } else {
    m_actionScheduler->resume();
  }

  InspectorInstrumentation::didLayout(cookie);

  m_nestedLayoutCount--;
#if ENABLE(ANDROID_OVERFLOW_SCROLL)
  // Reset to false each time we layout in case the overflow status changed.
  bool hasOverflowScroll = false;
  RenderObject* ownerRenderer = m_frame->ownerRenderer();
  if (ownerRenderer && ownerRenderer->isRenderIFrame()) {
    RenderLayer* layer = ownerRenderer->enclosingLayer();
    if (layer) {
      // Some sites use tiny iframes for loading so don't composite those.
      if (canHaveScrollbars() && layoutWidth() > 1 && layoutHeight() > 1)
        hasOverflowScroll = layoutWidth() < contentsWidth() || layoutHeight() < contentsHeight();
    }
  }

  if (RenderView* view = m_frame->contentRenderer()) {
    if (hasOverflowScroll != m_hasOverflowScroll) {
      if (hasOverflowScroll)
        enterCompositingMode();
      else {
        // We are leaving overflow mode so we need to update the layer
        // tree in case that is the reason we were composited.
        view->compositor()->scheduleCompositingLayerUpdate();
      }
    }
  }

  m_hasOverflowScroll = hasOverflowScroll;
#endif
}

// Sometimes (for plug-ins) we need to eagerly go into compositing mode.
void FrameView::enterCompositingMode() {
#if USE(ACCELERATED_COMPOSITING)
  if (RenderView* view = m_frame->contentRenderer()) {
    view->compositor()->enableCompositingMode();
    if (!needsLayout())
      view->compositor()->scheduleCompositingLayerUpdate();
  }
#endif
}

// Note that this gets called at painting time.
void FrameView::setIsOverlapped(bool isOverlapped) {
  if (isOverlapped == m_isOverlapped)
    return;

  m_isOverlapped = isOverlapped;
  updateCanBlitOnScrollRecursively();

#if USE(ACCELERATED_COMPOSITING)
  if (hasCompositedContentIncludingDescendants()) {
    // Overlap can affect compositing tests, so if it changes, we need to trigger
    // a layer update in the parent document.
    if (Frame* parentFrame = m_frame->tree()->parent()) {
      if (RenderView* parentView = parentFrame->contentRenderer()) {
        RenderLayerCompositor* compositor = parentView->compositor();
        compositor->setCompositingLayersNeedRebuild();
        compositor->scheduleCompositingLayerUpdate();
      }
    }

    if (RenderLayerCompositor::allowsIndependentlyCompositedFrames(this)) {
      // We also need to trigger reevaluation for this and all descendant frames,
      // since a frame uses compositing if any ancestor is compositing.
      for (Frame* frame = m_frame.get(); frame; frame = frame->tree()->traverseNext(m_frame.get())) {
        if (RenderView* view = frame->contentRenderer()) {
          RenderLayerCompositor* compositor = view->compositor();
          compositor->setCompositingLayersNeedRebuild();
          compositor->scheduleCompositingLayerUpdate();
        }
      }
    }
  }
#endif
}

void RenderLayerCompositor::scheduleCompositingLayerUpdate() {
  if (!m_updateCompositingLayersTimer.isActive())
    m_updateCompositingLayersTimer.startOneShot(0);
}

bool RenderLayerCompositor::compositingLayerUpdatePending() const {
  return m_updateCompositingLayersTimer.isActive();
}

void RenderLayerCompositor::updateCompositingLayersTimerFired(Timer<RenderLayerCompositor>*) {
  updateCompositingLayers();
}

// WebCore/platform/graphics/android/GraphicsLayerAndroid.cpp

bool GraphicsLayerAndroid::setChildren(const WTF::Vector<GraphicsLayer*>& children)

void GraphicsLayerAndroid::addChild(GraphicsLayer* childLayer)

void GraphicsLayerAndroid::addChildAtIndex(GraphicsLayer* childLayer, int index)

void GraphicsLayerAndroid::addChildBelow(GraphicsLayer* childLayer, GraphicsLayer* sibling)

void GraphicsLayerAndroid::addChildAbove(GraphicsLayer* childLayer, GraphicsLayer* sibling)

bool GraphicsLayerAndroid::replaceChild(GraphicsLayer* oldChild, GraphicsLayer* newChild)

void GraphicsLayerAndroid::removeFromParent()

void GraphicsLayerAndroid::setZPosition(float position)

...

{
  askForSync();
}

void GraphicsLayerAndroid::askForSync() {
  if (m_client)
    m_client->notifySyncRequired(this);
}

// WebCore/rendering/RenderLayerBacking.cpp

void RenderLayerBacking::notifySyncRequired(const GraphicsLayer*) {
  if (!renderer()->documentBeingDestroyed())
    compositor()->scheduleLayerFlush();
}

// WebCore/rendering/RenderLayerCompositor.h

void RenderLayerCompositor::notifySyncRequired(const GraphicsLayer*) {
  scheduleLayerFlush();
}

// WebCore/rendering/RenderLayerCompositor.cpp

void RenderLayerCompositor::scheduleLayerFlush() {
  page->chrome()->client()->scheduleCompositingLayerSync();
}
```

## documentDidBecomeActive 驱动

``` cpp
void Document::documentDidBecomeActive() {
  HashSet<Element*>::iterator end = m_documentActivationCallbackElements.end();
  for (HashSet<Element*>::iterator i = m_documentActivationCallbackElements.begin(); i != end; ++i)
    (*i)->documentDidBecomeActive();

#if USE(ACCELERATED_COMPOSITING)
  if (renderer())
    renderView()->didMoveOnscreen();
#endif

  ASSERT(m_frame);
  m_frame->loader()->client()->dispatchDidBecomeFrameset(isFrameSet());
}

// WebCore/rendering/RenderLayerCompositor.cpp

void RenderLayerCompositor::didMoveOnscreen() {
  RootLayerAttachment attachment = shouldPropagateCompositingToEnclosingFrame() ? RootLayerAttachedViaEnclosingFrame
                                                                                : RootLayerAttachedViaChromeClient;
  attachRootPlatformLayer(attachment);
}

void RenderLayerCompositor::attachRootPlatformLayer(RootLayerAttachment attachment) {
  switch (attachment) {
    case RootLayerAttachedViaChromeClient: {
      page->chrome()->client()->attachRootGraphicsLayer(frame, rootPlatformLayer());
      break;
    }
    case RootLayerAttachedViaEnclosingFrame: {
      // The layer will get hooked up via RenderLayerBacking::updateGraphicsLayerConfiguration()
      // for the frame's renderer in the parent document.
      scheduleNeedsStyleRecalc(m_renderView->document()->ownerElement());
      break;
    }
  }
}

// WebKit/gaia/WebCoreSupport/ChromeClientAndroid.cpp

void ChromeClientAndroid::attachRootGraphicsLayer(WebCore::Frame*, WebCore::GraphicsLayer* layer) {
  scheduleCompositingLayerSync();
}

// WebKit/gaia/WebCoreSupport/ChromeClientAndroid.cpp
void ChromeClientAndroid::scheduleCompositingLayerSync() {
  if (m_needsLayerSync)
    return;

  m_needsLayerSync = true;
  WebViewCore* webViewCore = WebViewCore::getWebViewCore(m_webFrame->page()->mainFrame()->view());
  if (webViewCore)
    webViewCore->layersDraw();
}

// WebKit/gaia/frameworks/webkit/WebViewCore.cpp

void WebViewCore::layersDraw() {
  mEventHub->sendMessage(gaia::Message::obtain(NULL, EventHub::WEBKIT_DRAW_LAYERS));
}

// WebKit/gaia/frameworks/webkit/inner/EventHub.cpp

void EventHub::EventHubHandler::handleMessage(const sp<gaia::Message>& msg) {
  case WEBKIT_DRAW:
    webViewCore->webkitDraw();
    break;

  case WEBKIT_DRAW_LAYERS:
    webViewCore->webkitDrawLayers();
    break;
}

// WebKit/gaia/frameworks/webkit/WebViewCore.cpp

void WebViewCore::webkitDrawLayers() {
  mDrawLayersIsScheduled = false;
  if (mDrawIsScheduled || mLastDrawData == NULL) {
    removeMessages(EventHub::WEBKIT_DRAW);
    webkitDraw();
    return;
  }

  // Directly update the layers we last passed to the UI side
  if (nativeUpdateLayers(mProxy->mNativeClass, mLastDrawData->mBaseLayer)) {
    // If anything more complex than position has been touched, let's do a full draw
    webkitDraw();
  }
}

void WebViewCore::webkitDraw() {
  mDrawIsScheduled = false;
  sp<DrawData> draw = new DrawData();
  draw->mBaseLayer = nativeRecordContent(draw->mInvalRegion, draw->mContentSize);
  if (draw->mBaseLayer == 0) {
    sp <WebView> spWebView = mWebView.promote();
    if (spWebView != NULL && !spWebView->isPaused()) {
      LOGV("webkitDraw abort, resending draw message");
      mEventHub->sendMessage(gaia::Message::obtain(NULL, EventHub::WEBKIT_DRAW));
    } else
      LOGV("webkitDraw abort, webview paused");
    return;
  }

  mLastDrawData = draw;
  webkitDraw(draw);
}
```

``` cpp
// WebKit/gaia/frameworks/webkit/WebViewCore.cpp

void WebViewCore::webkitDraw(const sp<DrawData>& draw) {
  sp <WebView> spWebView = mWebView.promote();
  if (spWebView != NULL) {
    draw->mFocusSizeChanged = nativeFocusBoundsChanged();
    draw->mViewSize = new GPoint(mCurrentViewWidth, mCurrentViewHeight);
    if (mSettings->getUseWideViewPort()) {
      draw->mMinPrefWidth = std::max(
          mViewportWidth == -1 ? WebView::DEFAULT_VIEWPORT_WIDTH
                               : (mViewportWidth == 0 ? mCurrentViewWidth
                                                      : mViewportWidth),
          nativeGetContentMinPrefWidth());
    }
    if (mInitialViewState != NULL) {
      draw->mViewState = mInitialViewState;
      mInitialViewState = NULL;
    }
    if (mFirstLayoutForNonStandardLoad) {
      draw->mFirstLayoutForNonStandardLoad = true;
      mFirstLayoutForNonStandardLoad = false;
    }
    if (DebugFlags::WEB_VIEW_CORE)
      LOGV("webkitDraw NEW_PICTURE_MSG_ID");

    sp<gaia::Message> message = gaia::Message::obtain(spWebView->mPrivateHandler,
                                                                WebView::NEW_PICTURE_MSG_ID, draw);
    SAFETY_ON_NULL_RETURN(message);
    message->sendToTarget();
  }
}

void PrivateHandler::handleMessage(const sp<gaia::Message>& msg) {
  case WebView::NEW_PICTURE_MSG_ID: {
    // called for new content
    sp<DrawData> draw = safe_cast<DrawData*>(msg->obj);
    webView->setNewPicture(draw, true);
    break;
  }
}

BaseLayerAndroid* WebViewCore::recordContent(SkRegion* region, SkIPoint* point) {
  float progress = (float) m_mainFrame->page()->progress()->estimatedProgress();
  m_progressDone = progress <= 0.0f || progress >= 1.0f;
  recordPictureSet(&m_content);
  return createBaseLayer(region);
}

void WebViewCore::recordPictureSet(PictureSet* content) {
  // Rebuild the pictureset (webkit repaint)
  rebuildPictureSet(content);

  updateFrameCache();
}

void WebViewCore::rebuildPictureSet(PictureSet* pictureSet) {
  WebCore::FrameView* view = m_mainFrame->view();
  size_t size = pictureSet->size();
  for (size_t index = 0; index < size; index++) {
    if (pictureSet->upToDate(index))
      continue;

    const SkIRect& inval = pictureSet->bounds(index);
    pictureSet->setPicture(index, rebuildPicture(inval));
  }

  pictureSet->validate(__FUNCTION__);
}

SkPicture* WebViewCore::rebuildPicture(const SkIRect& inval) {
  WebCore::FrameView* view = m_mainFrame->view();
  int width = view->contentsWidth();
  int height = view->contentsHeight();
  SkPicture* picture = new SkPicture();
  SkAutoPictureRecord arp(picture, width, height, PICT_RECORD_FLAGS);
  SkAutoMemoryUsageProbe mup(__FUNCTION__);
  SkCanvas* recordingCanvas = arp.getRecordingCanvas();

  WebCore::PlatformGraphicsContext pgc(recordingCanvas);
  WebCore::GraphicsContext gc(&pgc);
  IntPoint origin = view->minimumScrollPosition();
  // Line commented because of highlight problem
  // WebCore::IntRect drawArea(inval.fLeft + origin.x(), inval.fTop + origin.y(), inval.width(), inval.height());
  recordingCanvas->translate(-drawArea.x(), -drawArea.y());
  recordingCanvas->save();
  view->platformWidget()->draw(&gc, drawArea);
  m_rebuildInval.op(inval, SkRegion::kUnion_Op);
  return picture;
}

BaseLayerAndroid* WebViewCore::createBaseLayer(SkRegion* region) {
  BaseLayerAndroid* base = new BaseLayerAndroid();
  base->setContent(m_content);

  m_skipContentDraw = true;
  bool layoutSucceeded = layoutIfNeededRecursive(m_mainFrame);
  m_skipContentDraw = false;

#if USE(ACCELERATED_COMPOSITING)
  // We set the background color
  if (m_mainFrame && m_mainFrame->document()
      && m_mainFrame->document()->body()) {
    Document* document = m_mainFrame->document();
    RefPtr<RenderStyle> style = document->styleForElementIgnoringPendingStylesheets(document->body());
    if (style->hasBackground()) {
      Color color = style->visitedDependentColor(CSSPropertyBackgroundColor);
      if (color.isValid() && color.alpha() > 0)
        base->setBackgroundColor(color);
    }
  }

  // We update the layers
  ChromeClientAndroid* chromeC = static_cast<ChromeClientAndroid*>(m_mainFrame->page()->chrome()->client());
  GraphicsLayerAndroid* root = static_cast<GraphicsLayerAndroid*>(chromeC->layersSync());
  if (root) {
    LayerAndroid* copyLayer = new LayerAndroid(*root->contentLayer());
    base->addChild(copyLayer);
    copyLayer->unref();
    root->contentLayer()->clearDirtyRegion();
  }
#endif

  return base;
}
```