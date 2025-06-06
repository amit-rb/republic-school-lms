<script lang="ts">
  import { Send, Paperclip, Search, Phone, Video, MoreVertical, Smile, ArrowLeft, Menu } from 'lucide-svelte';
  import Layout from '../components/Layout.svelte';
  import { chatRooms, currentChatRoom, chatMessages } from '../stores/chat';
  import { currentUser } from '../stores/auth';
  import { onMount } from 'svelte';
  
  let messageInput = '';
  let searchQuery = '';
  let showEmojiPicker = false;
  let showRoomsList = false; // Controls mobile sidebar visibility
  let isMobile = false;
  
  // Check if device is mobile
  onMount(() => {
    const checkMobile = () => {
      isMobile = window.innerWidth < 768;
      // On desktop, always show rooms list
      if (!isMobile) {
        showRoomsList = true;
      }
    };
    
    checkMobile();
    window.addEventListener('resize', checkMobile);
    
    // Select first room by default if none selected
    if ($chatRooms.length > 0 && !$currentChatRoom) {
      selectRoom($chatRooms[0]);
    }
    
    return () => window.removeEventListener('resize', checkMobile);
  });
  
  // Filter chat rooms based on search
  $: filteredRooms = $chatRooms.filter(room => 
    room.name.toLowerCase().includes(searchQuery.toLowerCase())
  );
  
  // Get messages for current room
  $: roomMessages = $chatMessages.filter(message => 
    $currentChatRoom ? true : false // In real app, filter by room ID
  );
  
  function selectRoom(room: any) {
    currentChatRoom.set(room);
    // Mark messages as read
    room.unreadCount = 0;
    
    // On mobile, hide rooms list when a room is selected
    if (isMobile) {
      showRoomsList = false;
    }
  }
  
  function toggleRoomsList() {
    showRoomsList = !showRoomsList;
  }
  
  function sendMessage() {
    if (!messageInput.trim() || !$currentChatRoom) return;
    
    const newMessage = {
      id: Date.now().toString(),
      senderId: $currentUser?.id || '1',
      senderName: $currentUser?.name || 'You',
      senderAvatar: $currentUser?.avatar || '',
      content: messageInput.trim(),
      timestamp: new Date().toISOString(),
      type: 'text' as const,
      isRead: false
    };
    
    chatMessages.update(messages => [...messages, newMessage]);
    messageInput = '';
  }
  
  function handleKeyPress(event: KeyboardEvent) {
    if (event.key === 'Enter' && !event.shiftKey) {
      event.preventDefault();
      sendMessage();
    }
  }
  
  function formatTime(timestamp: string): string {
    return new Date(timestamp).toLocaleTimeString('en-US', {
      hour: '2-digit',
      minute: '2-digit'
    });
  }
  
  function formatDate(timestamp: string): string {
    const date = new Date(timestamp);
    const today = new Date();
    const yesterday = new Date(today);
    yesterday.setDate(yesterday.getDate() - 1);
    
    if (date.toDateString() === today.toDateString()) {
      return 'Today';
    } else if (date.toDateString() === yesterday.toDateString()) {
      return 'Yesterday';
    } else {
      return date.toLocaleDateString();
    }
  }
  
  function isNewDay(currentMessage: any, previousMessage: any): boolean {
    if (!previousMessage) return true;
    const currentDate = new Date(currentMessage.timestamp).toDateString();
    const previousDate = new Date(previousMessage.timestamp).toDateString();
    return currentDate !== previousDate;
  }
</script>

<Layout>
  <div class="h-[calc(100vh-4rem)] flex relative">
    <!-- Mobile Overlay -->
    {#if isMobile && showRoomsList}
      <div 
        class="fixed inset-0 bg-black bg-opacity-50 z-40 md:hidden"
        on:click={() => showRoomsList = false}
        role="button"
        tabindex="0"
        on:keydown={(e) => e.key === 'Enter' && (showRoomsList = false)}
        aria-label="Close chat rooms"
      ></div>
    {/if}

    <!-- Chat Rooms Sidebar -->
    <div class="
      {isMobile ? 'fixed inset-y-0 left-0 z-50 transform transition-transform duration-300' : 'relative'}
      {isMobile && !showRoomsList ? '-translate-x-full' : 'translate-x-0'}
      w-80 bg-white border-r border-gray-200 flex flex-col
      md:relative md:translate-x-0
    ">
      <!-- Header -->
      <div class="p-4 border-b border-gray-200">
        <div class="flex items-center justify-between mb-3">
          <h2 class="text-lg font-semibold text-gray-900">Messages</h2>
          {#if isMobile}
            <button
              class="p-2 rounded-lg hover:bg-gray-100 transition-colors duration-200 md:hidden"
              on:click={() => showRoomsList = false}
              aria-label="Close chat rooms"
            >
              <ArrowLeft class="w-5 h-5 text-gray-600" />
            </button>
          {/if}
        </div>
        
        <!-- Search -->
        <div class="relative">
          <Search class="absolute left-3 top-1/2 transform -translate-y-1/2 w-4 h-4 text-gray-400" />
          <input
            type="text"
            placeholder="Search conversations..."
            bind:value={searchQuery}
            class="w-full pl-10 pr-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-primary-500 focus:border-primary-500"
          />
        </div>
      </div>
      
      <!-- Chat Rooms List -->
      <div class="flex-1 overflow-y-auto">
        {#each filteredRooms as room}
          <button
            class="w-full p-4 text-left hover:bg-gray-50 border-b border-gray-100 transition-colors duration-200 {$currentChatRoom?.id === room.id ? 'bg-primary-50 border-primary-200' : ''}"
            on:click={() => selectRoom(room)}
          >
            <div class="flex items-start space-x-3">
              <div class="relative">
                {#if room.type === 'course'}
                  <div class="w-12 h-12 bg-primary-600 rounded-full flex items-center justify-center text-white font-bold">
                    {room.name.charAt(0)}
                  </div>
                {:else}
                  <img 
                    src={room.lastMessage?.senderAvatar} 
                    alt={room.name}
                    class="w-12 h-12 rounded-full object-cover"
                  />
                {/if}
                {#if room.unreadCount > 0}
                  <div class="absolute -top-1 -right-1 w-5 h-5 bg-error-500 text-white text-xs font-bold rounded-full flex items-center justify-center">
                    {room.unreadCount}
                  </div>
                {/if}
              </div>
              
              <div class="flex-1 min-w-0">
                <div class="flex items-center justify-between mb-1">
                  <h3 class="font-medium text-gray-900 truncate">{room.name}</h3>
                  {#if room.lastMessage}
                    <span class="text-xs text-gray-500">
                      {formatTime(room.lastMessage.timestamp)}
                    </span>
                  {/if}
                </div>
                
                {#if room.lastMessage}
                  <p class="text-sm text-gray-600 truncate">
                    {room.type === 'course' ? room.lastMessage.senderName + ': ' : ''}{room.lastMessage.content}
                  </p>
                {:else}
                  <p class="text-sm text-gray-400">No messages yet</p>
                {/if}
                
                <div class="flex items-center justify-between mt-1">
                  <span class="text-xs text-gray-400 capitalize">{room.type}</span>
                  {#if room.type === 'course'}
                    <div class="flex items-center space-x-1">
                      <div class="w-2 h-2 bg-success-500 rounded-full"></div>
                      <span class="text-xs text-gray-500">{room.participants.length} members</span>
                    </div>
                  {/if}
                </div>
              </div>
            </div>
          </button>
        {/each}
      </div>
    </div>

    <!-- Chat Area -->
    <div class="flex-1 flex flex-col min-w-0">
      {#if $currentChatRoom}
        <!-- Chat Header -->
        <div class="p-4 border-b border-gray-200 bg-white">
          <div class="flex items-center justify-between">
            <div class="flex items-center space-x-3">
              <!-- Mobile Menu Button -->
              {#if isMobile}
                <button
                  class="p-2 rounded-lg hover:bg-gray-100 transition-colors duration-200 md:hidden"
                  on:click={toggleRoomsList}
                  aria-label="Open chat rooms"
                >
                  <Menu class="w-5 h-5 text-gray-600" />
                </button>
              {/if}
              
              {#if $currentChatRoom.type === 'course'}
                <div class="w-10 h-10 bg-primary-600 rounded-full flex items-center justify-center text-white font-bold">
                  {$currentChatRoom.name.charAt(0)}
                </div>
              {:else}
                <img 
                  src={$currentChatRoom.lastMessage?.senderAvatar} 
                  alt={$currentChatRoom.name}
                  class="w-10 h-10 rounded-full object-cover"
                />
              {/if}
              
              <div class="min-w-0 flex-1">
                <h3 class="font-semibold text-gray-900 truncate">{$currentChatRoom.name}</h3>
                <p class="text-sm text-gray-500">
                  {#if $currentChatRoom.type === 'course'}
                    {$currentChatRoom.participants.length} members
                  {:else}
                    Online
                  {/if}
                </p>
              </div>
            </div>
            
            <div class="flex items-center space-x-2">
              {#if $currentChatRoom.type === 'direct'}
                <button class="p-2 rounded-lg hover:bg-gray-100 transition-colors duration-200 hidden sm:block">
                  <Phone class="w-5 h-5 text-gray-600" />
                </button>
                <button class="p-2 rounded-lg hover:bg-gray-100 transition-colors duration-200 hidden sm:block">
                  <Video class="w-5 h-5 text-gray-600" />
                </button>
              {/if}
              <button class="p-2 rounded-lg hover:bg-gray-100 transition-colors duration-200">
                <MoreVertical class="w-5 h-5 text-gray-600" />
              </button>
            </div>
          </div>
        </div>
        
        <!-- Messages Area -->
        <div class="flex-1 overflow-y-auto p-4 space-y-4 bg-gray-50">
          {#each roomMessages as message, index}
            {#if isNewDay(message, roomMessages[index - 1])}
              <div class="flex justify-center">
                <span class="px-3 py-1 bg-gray-200 text-gray-600 text-xs rounded-full">
                  {formatDate(message.timestamp)}
                </span>
              </div>
            {/if}
            
            <div class="flex items-start space-x-3 {message.senderId === $currentUser?.id ? 'flex-row-reverse space-x-reverse' : ''}">
              <img 
                src={message.senderAvatar} 
                alt={message.senderName}
                class="w-8 h-8 rounded-full object-cover flex-shrink-0"
              />
              
              <div class="flex flex-col {message.senderId === $currentUser?.id ? 'items-end' : 'items-start'} max-w-xs sm:max-w-sm lg:max-w-md">
                {#if $currentChatRoom.type === 'course' && message.senderId !== $currentUser?.id}
                  <span class="text-xs text-gray-500 mb-1">{message.senderName}</span>
                {/if}
                
                <div class="px-4 py-2 rounded-lg {message.senderId === $currentUser?.id ? 'bg-primary-600 text-white' : 'bg-white text-gray-900'} shadow-sm break-words">
                  <p class="text-sm">{message.content}</p>
                </div>
                
                <span class="text-xs text-gray-400 mt-1">
                  {formatTime(message.timestamp)}
                </span>
              </div>
            </div>
          {/each}
        </div>
        
        <!-- Message Input -->
        <div class="p-4 bg-white border-t border-gray-200">
          <div class="flex items-end space-x-3">
            <button class="p-2 rounded-lg hover:bg-gray-100 transition-colors duration-200 flex-shrink-0">
              <Paperclip class="w-5 h-5 text-gray-600" />
            </button>
            
            <div class="flex-1 relative">
              <textarea
                bind:value={messageInput}
                on:keydown={handleKeyPress}
                placeholder="Type a message..."
                rows="1"
                class="w-full px-4 py-2 pr-12 border border-gray-300 rounded-lg focus:ring-2 focus:ring-primary-500 focus:border-primary-500 resize-none"
                style="min-height: 40px; max-height: 120px;"
              ></textarea>
              
              <button 
                class="absolute right-2 top-1/2 transform -translate-y-1/2 p-1 rounded-lg hover:bg-gray-100 transition-colors duration-200"
                on:click={() => showEmojiPicker = !showEmojiPicker}
              >
                <Smile class="w-4 h-4 text-gray-600" />
              </button>
            </div>
            
            <button 
              class="p-2 bg-primary-600 text-white rounded-lg hover:bg-primary-700 transition-colors duration-200 disabled:opacity-50 flex-shrink-0"
              on:click={sendMessage}
              disabled={!messageInput.trim()}
            >
              <Send class="w-5 h-5" />
            </button>
          </div>
        </div>
      {:else}
        <!-- No Chat Selected -->
        <div class="flex-1 flex items-center justify-center bg-gray-50">
          <div class="text-center">
            {#if isMobile}
              <button
                class="mb-4 btn btn-primary"
                on:click={toggleRoomsList}
              >
                <Menu class="w-4 h-4 mr-2" />
                Select a conversation
              </button>
            {:else}
              <div class="w-16 h-16 bg-gray-200 rounded-full flex items-center justify-center mx-auto mb-4">
                <Send class="w-8 h-8 text-gray-400" />
              </div>
              <h3 class="text-lg font-medium text-gray-900 mb-2">Select a conversation</h3>
              <p class="text-gray-500">Choose a chat room to start messaging</p>
            {/if}
          </div>
        </div>
      {/if}
    </div>
  </div>
</Layout>