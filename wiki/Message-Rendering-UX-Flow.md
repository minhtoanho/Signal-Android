# Message Rendering UX Flow

> **This document explains how Signal-Android handles conversation message rendering, including large conversations, real-time message insertion, scroll position management, and edge cases like time gaps.**

## Table of Contents

1. [Overview](#1-overview)
2. [Architecture Layers](#2-architecture-layers)
3. [Message Loading and Windowing](#3-message-loading-and-windowing)
4. [Real-Time Message Handling](#4-real-time-message-handling)
5. [Scroll Position Management](#5-scroll-position-management)
6. [Edge Cases and Solutions](#6-edge-cases-and-solutions)
7. [Complete Flow Diagrams](#7-complete-flow-diagrams)

---

## 1. Overview

### Key Concepts

| Concept | Purpose | Location |
|---------|---------|----------|
| **ConversationFragment** | Main UI component for chat thread | `ConversationFragment.kt:110-5245` |
| **ConversationAdapterV2** | RecyclerView adapter for messages | `ConversationAdapterV2.kt:70-728` |
| **ConversationDataSource** | Paged data source for loading messages | `ConversationDataSource.kt:50-241` |
| **PagingConfig** | Configuration for window size and buffering | `ConversationRepository.kt:157-160` |
| **ScrollToPositionDelegate** | Manages scroll position requests | `ScrollToPositionDelegate.kt:27-219` |
| **ConversationScrollButtonState** | State for scroll buttons and unread indicators | `ConversationScrollButtonState.kt:11-20` |
| **ConversationLayoutManager** | Custom LinearLayoutManager for conversations | `ConversationLayoutManager.kt` |

### Performance Characteristics

| Metric | Value | Code Reference |
|--------|-------|----------------|
| **Page Size** | 25 messages | `ConversationRepository.kt:157` |
| **Buffer Pages** | 2 pages (50 messages) | `ConversationRepository.kt:158` |
| **Smooth Scroll Threshold** | 25 items | `ScrollToPositionDelegate.kt:36` |
| **Scroll Animation Threshold** | 50 items | `ScrollToPositionDelegate.kt:37` |

---

## 2. Architecture Layers

```
┌─────────────────────────────────────────────────────────────────────┐
│                    MESSAGE RENDERING ARCHITECTURE                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐ │
│  │    UI LAYER     │    │  VIEWMODEL      │    │   REPOSITORY    │ │
│  ├─────────────────┤    ├─────────────────┤    ├─────────────────┤ │
│  │ ConversationFrag│───▶│ ConversationVM  │───▶│ ConversationRepo│ │
│  │ :110-5245       │    │ :110-826        │    │ :116-932        │ │
│  │                 │    │                 │    │                 │ │
│  │ ConversationAdap│    │ scrollButtonState│    │ getThreadState  │ │
│  │ :70-728         │    │ :126-133        │    │ :141-167        │ │
│  └─────────────────┘    └─────────────────┘    └─────────────────┘ │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │                    PAGING LAYER (lib/paging)                    ││
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         ││
│  │  │ PagedData    │  │PagingController│  │PagingConfig │         ││
│  │  │              │  │ :4-11         │  │ :157-160     │         ││
│  │  └──────────────┘  └──────────────┘  └──────────────┘         ││
│  └─────────────────────────────────────────────────────────────────┘│
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │                    DATA SOURCE LAYER                            ││
│  │  ConversationDataSource.kt:50-241                               ││
│  │  - load(start, length) :96-159                                  ││
│  │  - size() :72-82                                                ││
│  └─────────────────────────────────────────────────────────────────┘│
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │                    DATABASE LAYER                               ││
│  │  MessageTable.kt - getConversation()                            ││
│  │  DatabaseObserver.java - notify message inserts                 ││
│  └─────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Message Loading and Windowing

### 3.1 Paging Configuration

```kotlin
// File: ConversationRepository.kt
// Lines: 157-160

val config = PagingConfig.Builder()
  .setPageSize(25)          // Load 25 messages per page
  .setBufferPages(2)        // Keep 2 extra pages buffered (50 messages)
  .setStartIndex(max(metadata.getStartPosition(), 0))
  .build()
```

**Business Logic:**
- **PageSize = 25**: Optimal balance between database query time and memory usage
- **BufferPages = 2**: Ensures smooth scrolling by pre-loading adjacent pages
- **Total Window = 75 messages**: 25 visible + 50 buffered = smooth UX

### 3.2 Data Source Loading

```
┌─────────────────────────────────────────────────────────────────────┐
│          MESSAGE LOADING FLOW                                        │
│          ConversationDataSource.load()                              │
│          File: ConversationDataSource.kt:96-159                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  TRIGGER: PagingController.onDataNeededAroundIndex()               │
│                                                                     │
│  STEP 1: Query Database                                            │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ // Load messages from database                               │  │
│  │ MessageTable.mmsReaderFor(                                   │  │
│  │   SignalDatabase.messages.getConversation(                   │  │
│  │     threadId,                                                │  │
│  │     start.toLong(),    // Offset in conversation             │  │
│  │     length.toLong(),   // Number of messages to load         │  │
│  │     filterCollapsed = true                                   │  │
│  │   )                                                          │  │
│  │ )                                                            │  │
│  │                                                              │  │
│  │ Reference: ConversationDataSource.kt:100                     │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  STEP 2: Fetch Extra Data (Attachments, Mentions, etc.)            │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ val extraData = MessageDataFetcher.fetch(records, threadRecipient)│
│  │ records = MessageDataFetcher.updateModelsWithData(records, extraData)│
│  │                                                              │  │
│  │ Reference: ConversationDataSource.kt:121-124                 │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  STEP 3: Convert to UI Models                                      │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ val messages = records.map { record ->                        │  │
│  │   ConversationMessageFactory.createWithUnresolvedData(       │  │
│  │     localContext, record, displayBody, mentions, ...         │  │
│  │   ).toMappingModel()                                         │  │
│  │ }                                                            │  │
│  │                                                              │  │
│  │ Reference: ConversationDataSource.kt:132-142                 │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  OUTPUT: List<ConversationElement> for adapter                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.3 Large Conversation Handling

**Problem:** User opens conversation with 10,000+ messages.

**Solution:**
1. **Initial Load**: Load only 25 messages around the requested position
2. **Lazy Loading**: Additional pages load on-demand as user scrolls
3. **Database Observer**: Invalidates paging when new messages arrive

```kotlin
// File: ConversationRepository.kt
// Lines: 141-167

fun getConversationThreadState(threadId: Long, requestedStartPosition: Int): Single<ConversationThreadState> {
  return Single.fromCallable {
    val metadata = oldConversationRepository.getConversationData(threadId, recipient, requestedStartPosition)
    
    // Create data source with KNOWN size to avoid full count query
    val dataSource = ConversationDataSource(
      localContext,
      threadId,
      messageRequestData,
      metadata.showUniversalExpireTimerMessage,
      metadata.threadSize  // Pass known size to avoid expensive COUNT query
    )
    
    // Configure paging with start position
    val config = PagingConfig.Builder()
      .setPageSize(25)
      .setBufferPages(2)
      .setStartIndex(max(metadata.getStartPosition(), 0))  // Jump to specific position
      .build()
    
    ConversationThreadState(
      items = PagedData.createForObservable(dataSource, config),
      meta = metadata
    )
  }.subscribeOn(Schedulers.io())
}
```

---

## 4. Real-Time Message Handling

### 4.1 Message Reception Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│          REAL-TIME MESSAGE RECEPTION FLOW                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  STEP 1: WebSocket Receives Message                                │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ IncomingMessageObserver.processEnvelope()                     │  │
│  │ - Receives encrypted envelope from WebSocket                 │  │
│  │ - Decrypts using MessageDecryptor                            │  │
│  │                                                              │  │
│  │ File: IncomingMessageObserver.kt                             │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  STEP 2: Queue Processing Job                                      │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ PushProcessMessageJob                                        │  │
│  │ - Validates message                                          │  │
│  │ - Processes content via MessageContentProcessor              │  │
│  │ - Inserts into database                                      │  │
│  │                                                              │  │
│  │ File: PushProcessMessageJob.kt                               │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  STEP 3: Database Observer Notifies                                │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ DatabaseObserver.notifyMessageInsertListeners(threadId)      │  │
│  │ - Called after message inserted                              │  │
│  │ - Notifies all registered listeners                          │  │
│  │                                                              │  │
│  │ File: DatabaseObserver.java                                  │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  STEP 4: ViewModel Receives Notification                           │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ ConversationViewModel                                        │  │
│  │ - Registered as DatabaseObserver listener                    │  │
│  │ - Invalidates paging data on message insert                  │  │
│  │ - Updates unread count                                       │  │
│  │                                                              │  │
│  │ File: ConversationViewModel.kt                               │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  STEP 5: UI Updates                                                │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ ConversationFragment                                         │  │
│  │ - Receives new ConversationThreadState                       │  │
│  │ - Submits new list to adapter                                │  │
│  │ - Adapter animates new message into view                     │  │
│  │ - Scroll position maintained or scrolled to bottom           │  │
│  │                                                              │  │
│  │ File: ConversationFragment.kt:1120-1130                      │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.2 Scroll Behavior on New Message

```kotlin
// File: ConversationScrollButtonState.kt
// Lines: 11-20

data class ConversationScrollButtonState(
  val hideScrollButtonsForReactionOverlay: Boolean = false,
  val showScrollButtonsForScrollPosition: Boolean = false,
  val willScrollToBottomOnNewMessage: Boolean = true,  // KEY DECISION
  val unreadCount: Int = 0,
  val hasMentions: Boolean = false
) {
  val showScrollButtons: Boolean
    get() = !hideScrollButtonsForReactionOverlay && 
           (showScrollButtonsForScrollPosition || 
            (!willScrollToBottomOnNewMessage && unreadCount > 0))
}
```

**Decision Matrix:**

| User Position | New Message Arrives | Action | Code Reference |
|---------------|---------------------|--------|----------------|
| At bottom | Yes | Auto-scroll to show new message | `ConversationFragment.kt:3242` |
| Scrolled up | Yes | Show scroll button with unread count | `ConversationScrollButtonState.kt:19` |
| Scrolled up | Multiple messages | Accumulate unread count, show button | `ConversationViewModel.kt:132-133` |
| At bottom | Mention | Scroll to show + highlight | `ConversationFragment.kt:858` |

---

## 5. Scroll Position Management

### 5.1 ScrollToPositionDelegate

```kotlin
// File: ScrollToPositionDelegate.kt
// Lines: 27-95

class ScrollToPositionDelegate private constructor(
  private val recyclerView: RecyclerView,
  canJumpToPosition: (Int) -> Boolean,
  mapToTruePosition: (Int) -> Int,
  private val disposables: CompositeDisposable
) : Disposable by disposables {
  
  companion object {
    private const val SMOOTH_SCROLL_THRESHOLD = 25     // Smooth scroll if < 25 items away
    private const val SCROLL_ANIMATION_THRESHOLD = 50  // Animate if < 50 items away
  }
  
  /**
   * Request scroll to specific position
   * @param position Target position (0 = bottom/newest)
   * @param smooth Use smooth scroll if within threshold
   */
  fun requestScrollPosition(position: Int, smooth: Boolean = true, scrollStrategy: ScrollStrategy = DefaultScrollStrategy) {
    scrollPositionRequested.onNext(ScrollToPositionRequest(position, smooth, scrollStrategy))
  }
  
  /**
   * Scroll to bottom (position 0)
   */
  fun resetScrollPosition() {
    requestScrollPosition(0, true)
  }
}
```

### 5.2 Scroll Strategy Implementations

```kotlin
// File: ScrollToPositionDelegate.kt
// Lines: 163-200

/**
 * Default: Pin to "top" of recycler (bottom visually due to reverseLayout)
 */
object DefaultScrollStrategy : ScrollStrategy {
  override fun performScroll(recyclerView: RecyclerView, layoutManager: LinearLayoutManager, position: Int, smooth: Boolean) {
    val offset = when {
      position == 0 -> 0
      layoutManager.reverseLayout -> recyclerView.height
      else -> 0
    }
    
    if (smooth && position == 0 && layoutManager.findFirstVisibleItemPosition() < SMOOTH_SCROLL_THRESHOLD) {
      recyclerView.smoothScrollToPosition(position)
    } else {
      layoutManager.scrollToPositionWithOffset(position, offset)
    }
  }
}

/**
 * Jump to position but ensure content is visible on screen
 */
object JumpToPositionStrategy : ScrollStrategy {
  override fun performScroll(recyclerView: RecyclerView, layoutManager: LinearLayoutManager, position: Int, smooth: Boolean) {
    if (abs(layoutManager.findFirstVisibleItemPosition() - position) < SCROLL_ANIMATION_THRESHOLD) {
      val child: View? = layoutManager.findViewByPosition(position)
      if (child == null || !layoutManager.isViewPartiallyVisible(child, true, false)) {
        layoutManager.scrollToPositionWithOffset(position, recyclerView.height / 3)
      }
    } else {
      layoutManager.scrollToPositionWithOffset(position, recyclerView.height / 3)
    }
  }
}
```

### 5.3 Scroll Button Logic

```
┌─────────────────────────────────────────────────────────────────────┐
│          SCROLL BUTTON VISIBILITY LOGIC                             │
│          File: ConversationFragment.kt:1850-1860                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  TRIGGER: scrollButtonState Flow emits new state                   │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ private fun presentScrollButtons(state: ConversationScrollButtonState) {│
│  │   binding.scrollToBottom.setUnreadCount(state.unreadCount)   │  │
│  │   binding.scrollToBottom.isShown = state.showScrollButtons   │  │
│  │ }                                                            │  │
│  │                                                              │  │
│  │ Reference: ConversationFragment.kt:1850-1855                 │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  SHOW SCROLL BUTTON WHEN:                                          │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ showScrollButtons =                                          │  │
│  │   !hideScrollButtonsForReactionOverlay &&                    │  │
│  │   (                                                          │  │
│  │     showScrollButtonsForScrollPosition ||                    │  │
│  │     (!willScrollToBottomOnNewMessage && unreadCount > 0)    │  │
│  │   )                                                          │  │
│  │                                                              │  │
│  │ Reference: ConversationScrollButtonState.kt:18-19            │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  CLICK HANDLER:                                                    │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ binding.scrollToBottom.setOnClickListener {                   │  │
│  │   scrollToPositionDelegate.resetScrollPosition()              │  │
│  │ }                                                            │  │
│  │                                                              │  │
│  │ Reference: ConversationFragment.kt:1250-1253                 │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 6. Edge Cases and Solutions

### 6.1 Edge Case: User Viewing Old Messages (Large Gap)

**Scenario:** User scrolled up to view messages from 6 months ago. 100 new messages arrive via WebSocket.

```
┌─────────────────────────────────────────────────────────────────────┐
│          EDGE CASE: LARGE TIME GAP                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  PROBLEM:                                                           │
│  - User is viewing messages from position 5000                      │
│  - 100 new messages arrive (now position 5100 is oldest)           │
│  - Gap between visible messages and newest is 100 messages          │
│                                                                     │
│  SOLUTION:                                                          │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ 1. DO NOT auto-scroll to bottom                              │  │
│  │    - willScrollToBottomOnNewMessage = false                  │  │
│  │    - User stays at position 5000                             │  │
│  │                                                              │  │
│  │ 2. Show scroll button with unread count                      │  │
│  │    - unreadCount = 100                                       │  │
│  │    - showScrollButtons = true                                │  │
│  │                                                              │  │
│  │ 3. User clicks scroll button                                 │  │
│  │    - resetScrollPosition() called                            │  │
│  │    - Jump to position 0 (newest)                             │  │
│  │    - NOT smooth scroll (too far)                             │  │
│  │                                                              │  │
│  │ Reference: ConversationScrollButtonState.kt:14-19            │  │
│  │            ScrollToPositionDelegate.kt:92-95                  │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  CODE:                                                              │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ // Update scroll button state                                │  │
│  │ viewModel.setShowScrollButtonsForScrollPosition(             │  │
│  │   showScrollButtons = true,                                  │  │
│  │   willScrollToBottomOnNewMessage = false                     │  │
│  │ )                                                            │  │
│  │                                                              │  │
│  │ Reference: ConversationFragment.kt:3240                      │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.2 Edge Case: Jump to Newest (After Scrolling Up)

**Scenario:** User clicked "scroll to bottom" button after viewing old messages.

```
┌─────────────────────────────────────────────────────────────────────┐
│          JUMP TO NEWEST FLOW                                        │
│          File: ScrollToPositionDelegate.kt:92-95                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  STEP 1: User Clicks Scroll Button                                 │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ binding.scrollToBottom.setOnClickListener {                   │  │
│  │   scrollToPositionDelegate.resetScrollPosition()              │  │
│  │ }                                                            │  │
│  │                                                              │  │
│  │ Reference: ConversationFragment.kt:1250-1252                 │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  STEP 2: Determine Scroll Strategy                                 │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ // Check distance from current position to target            │  │
│  │ if (smooth && position == 0 &&                               │  │
│  │     layoutManager.findFirstVisibleItemPosition() < 25) {     │  │
│  │   // Close to bottom: smooth scroll                          │  │
│  │   recyclerView.smoothScrollToPosition(0)                     │  │
│  │ } else {                                                     │  │
│  │   // Far from bottom: instant jump                           │  │
│  │   layoutManager.scrollToPositionWithOffset(0, 0)              │  │
│  │ }                                                            │  │
│  │                                                              │  │
│  │ Reference: ScrollToPositionDelegate.kt:178-182               │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  STEP 3: Reset Scroll State                                        │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ // Reset unread count and button visibility                  │  │
│  │ viewModel.setShowScrollButtonsForScrollPosition(             │  │
│  │   showScrollButtons = false,                                 │  │
│  │   willScrollToBottomOnNewMessage = true                      │  │
│  │ )                                                            │  │
│  │                                                              │  │
│  │ Reference: ConversationFragment.kt:3238                      │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.3 Edge Case: Very Large Conversation (10,000+ messages)

**Scenario:** User opens conversation with massive history.

```
┌─────────────────────────────────────────────────────────────────────┐
│          LARGE CONVERSATION OPTIMIZATION                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  PROBLEM:                                                           │
│  - 10,000+ messages in conversation                                │
│  - Cannot load all into memory                                     │
│  - COUNT(*) query is expensive                                     │
│                                                                     │
│  SOLUTIONS:                                                         │
│                                                                     │
│  1. PASS KNOWN SIZE TO DATA SOURCE                                 │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ // In ConversationDataSource constructor                     │  │
│  │ private var baseSize: Int  // Passed from metadata           │  │
│  │                                                              │  │
│  │ override fun size(): Int {                                   │  │
│  │   synchronized(this) {                                       │  │
│  │     if (baseSize != -1) {                                    │  │
│  │       val size = baseSize                                    │  │
│  │       baseSize = -1  // Use cached size once                 │  │
│  │       return size                                            │  │
│  │     }                                                        │  │
│  │   }                                                          │  │
│  │   // Fallback to database query only if needed               │  │
│  │   return SignalDatabase.messages.getMessageCountForThread(threadId)│
│  │ }                                                            │  │
│  │                                                              │  │
│  │ Reference: ConversationDataSource.kt:84-94                   │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  2. LAZY LOAD WITH PAGING                                          │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ // Only load 25 messages initially                           │  │
│  │ // Additional pages load on-demand as user scrolls           │  │
│  │ PagingConfig.Builder()                                       │  │
│  │   .setPageSize(25)        // Small initial load              │  │
│  │   .setBufferPages(2)      // Keep 50 extra buffered          │  │
│  │   .setStartIndex(position) // Jump to specific position      │  │
│  │                                                              │  │
│  │ Reference: ConversationRepository.kt:157-160                 │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  3. DIFFUTIL FOR EFFICIENT UPDATES                                 │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ // Adapter uses DiffUtil to only update changed items        │  │
│  │ class ConversationAdapterV2 : PagingMappingAdapter<Key>()   │  │
│  │                                                              │  │
│  │ // DiffUtil calculates minimal changes                       │  │
│  │ // Only new/changed messages are re-bound                    │  │
│  │                                                              │  │
│  │ Reference: ConversationAdapterV2.kt:70-79                    │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.4 Edge Case: Message Insertion While Scrolling

**Scenario:** User is actively scrolling when new message arrives.

```
┌─────────────────────────────────────────────────────────────────────┐
│          SCROLL STATE HANDLING                                      │
│          File: ScrollToPositionDelegate.kt:140-142                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  CODE:                                                              │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ private fun handleScrollPositionRequest(request: Request) {  │  │
│  │   // ...                                                     │  │
│  │                                                              │  │
│  │   // DO NOT scroll if user is actively dragging             │  │
│  │   if (recyclerView.scrollState == RecyclerView.SCROLL_STATE_DRAGGING) {│
│  │     return                                                   │  │
│  │   }                                                          │  │
│  │                                                              │  │
│  │   // Only scroll if user is idle or settling                │  │
│  │   performScroll(...)                                         │  │
│  │ }                                                            │  │
│  │                                                              │  │
│  │ Reference: ScrollToPositionDelegate.kt:140-142               │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  SCROLL STATES:                                                     │
│  - SCROLL_STATE_IDLE: Not scrolling                                │
│  - SCROLL_STATE_DRAGGING: User finger on screen                    │
│  - SCROLL_STATE_SETTLING: Fling animation in progress              │
│                                                                     │
│  BEHAVIOR:                                                          │
│  - If DRAGGING: Skip scroll, show button instead                   │
│  - If IDLE/SETTLING: Allow scroll if appropriate                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 7. Complete Flow Diagrams

### 7.1 Complete Message Rendering Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│          COMPLETE FLOW: USER OPENS CONVERSATION                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  USER ACTION: Open conversation with 10,000 messages               │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 1. ConversationActivity.kt                                  │   │
│  │    - Creates ConversationFragment                           │   │
│  │    - Passes threadId                                        │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                            │                                        │
│                            ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 2. ConversationFragment.onViewCreated()                     │   │
│  │    File: ConversationFragment.kt:2100-2200                  │   │
│  │    - Initializes ConversationViewModel                      │   │
│  │    - Sets up RecyclerView with ConversationLayoutManager    │   │
│  │    - Creates ScrollToPositionDelegate                       │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                            │                                        │
│                            ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 3. ConversationViewModel.loadConversationThreadState()      │   │
│  │    File: ConversationViewModel.kt:200-250                   │   │
│  │    - Calls repository.getConversationThreadState()          │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                            │                                        │
│                            ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 4. ConversationRepository.getConversationThreadState()      │   │
│  │    File: ConversationRepository.kt:141-167                  │   │
│  │    - Gets thread metadata (size, start position)            │   │
│  │    - Creates ConversationDataSource with known size         │   │
│  │    - Configures PagingConfig(pageSize=25, bufferPages=2)    │   │
│  │    - Returns ConversationThreadState                        │   │
│  └─────────────────────────────────────────────────────┬───────┘   │
│                            │                                        │
│                            ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 5. ConversationDataSource.load(start=0, length=25)          │   │
│  │    File: ConversationDataSource.kt:96-159                   │   │
│  │    - Queries database for 25 messages at start position     │   │
│  │    - Fetches extra data (attachments, mentions)             │   │
│  │    - Converts to ConversationMessage UI models              │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                            │                                        │
│                            ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 6. ConversationAdapterV2.submitList()                       │   │
│  │    File: ConversationAdapterV2.kt                           │   │
│  │    - Receives list of 25 messages                           │   │
│  │    - Uses DiffUtil to calculate changes                     │   │
│  │    - Binds visible items to ViewHolders                     │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                            │                                        │
│                            ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 7. ScrollToPositionDelegate.notifyListCommitted()           │   │
│  │    File: ScrollToPositionDelegate.kt:118-120                │   │
│  │    - Signals that list is ready                             │   │
│  │    - Executes any pending scroll requests                   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  OUTPUT: User sees 25 most recent messages, can scroll for more    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 7.2 Flow: Real-Time Message Arrives

```
┌─────────────────────────────────────────────────────────────────────┐
│          FLOW: NEW MESSAGE ARRIVES WHILE VIEWING CONVERSATION      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 1. WebSocket receives new message                           │   │
│  │    IncomingMessageObserver.processEnvelope()                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                            │                                        │
│                            ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 2. Message decrypted and processed                          │   │
│  │    PushProcessMessageJob → MessageContentProcessor           │   │
│  │    → Insert into MessageTable                               │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                            │                                        │
│                            ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 3. DatabaseObserver notifies listeners                      │   │
│  │    DatabaseObserver.notifyMessageInsertListeners(threadId)  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                            │                                        │
│                            ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 4. ViewModel receives notification                          │   │
│  │    ConversationViewModel.onMessageInsert()                  │   │
│  │    - Invalidates PagedData                                  │   │
│  │    - Updates unread count if scrolled up                    │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                            │                                        │
│                            ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 5. Check scroll position                                    │   │
│  │    if (willScrollToBottomOnNewMessage) {                    │   │
│  │      // Auto-scroll to show new message                     │   │
│  │      scrollToPositionDelegate.resetScrollPosition()          │   │
│  │    } else {                                                 │   │
│  │      // Show scroll button with unread count                │   │
│  │      updateScrollButtonState(unreadCount++)                  │   │
│  │    }                                                        │   │
│  │                                                             │   │
│  │    Reference: ConversationScrollButtonState.kt:14-19        │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                            │                                        │
│                            ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 6. UI updates                                               │   │
│  │    - New message appears in RecyclerView                    │   │
│  │    - Either auto-scrolled or button shown                   │   │
│  │    - Unread indicator updated                               │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Key Files Reference

| File | Purpose | Key Lines |
|------|---------|-----------|
| `ConversationFragment.kt` | Main conversation UI | 110-5245, scroll handling: 1850-1860, scrollToBottom: 3173-3174 |
| `ConversationAdapterV2.kt` | Message adapter | 70-728 |
| `ConversationViewModel.kt` | UI state management | 110-826, scrollButtonState: 126-133 |
| `ConversationRepository.kt` | Data loading | 116-932, paging config: 157-160 |
| `ConversationDataSource.kt` | Paged data source | 50-241, load(): 96-159 |
| `ScrollToPositionDelegate.kt` | Scroll position management | 27-219, thresholds: 36-37 |
| `ConversationScrollButtonState.kt` | Scroll button state | 11-20 |
| `ConversationLayoutManager.kt` | Custom layout manager | scrollToPositionWithOffset |
| `DatabaseObserver.java` | Real-time notifications | notifyMessageInsertListeners() |
| `IncomingMessageObserver.kt` | WebSocket message reception | processEnvelope() |
| `PushProcessMessageJob.kt` | Message processing job | |

---

## Summary

### Message Loading Strategy
- **Page size**: 25 messages per load
- **Buffer**: 2 extra pages (50 messages) for smooth scrolling
- **Lazy loading**: On-demand as user scrolls
- **Size caching**: Pass known size to avoid expensive COUNT queries

### Real-Time Message Handling
- **DatabaseObserver pattern**: UI notified of message inserts
- **Scroll state aware**: Different behavior when user is at bottom vs scrolled up
- **Unread count**: Accumulates when scrolled up, shown on scroll button

### Scroll Position Management
- **ScrollToPositionDelegate**: Centralized scroll position requests
- **Smart scrolling**: Smooth for nearby, instant jump for far positions
- **Thresholds**: Smooth < 25 items, animate < 50 items
- **User dragging respect**: Never auto-scroll while user is actively scrolling

### Edge Case Solutions
| Edge Case | Solution | Code Reference |
|-----------|----------|----------------|
| Large gap (old messages) | Show scroll button with unread count | `ConversationScrollButtonState.kt:19` |
| Jump to newest | Instant scroll, no animation | `ScrollToPositionDelegate.kt:181` |
| 10,000+ messages | Lazy paging, size caching | `ConversationDataSource.kt:84-94` |
| Scrolling while message arrives | Skip auto-scroll, show button | `ScrollToPositionDelegate.kt:140-142` |