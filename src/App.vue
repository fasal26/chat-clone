<template>
  <div class="chat-container" ref="chatContainer" @click="hideDropdown">
    <div class="chat-messages" ref="chatMessages">
      <div v-for="(key, index) in Object.keys(messages)" :key="index" class="message">
        <span v-for="(msgObj) in messages[key]" :class="{ 'tag': msgObj.type == 'mention' }">
          {{ msgObj.msg }}
        </span>
      </div>
    </div>
    <div class="chat-input" ref="chatInput">
      <div class="text-area-container">
        <div id="hidden-div" ref="msgCopy"></div>
        <textarea 
          ref="textarea"
          @input="handleInput"
          v-model="newMessage" 
          placeholder="Type a message..."
          @keyup="handleKeydown"
        ></textarea>
      </div>
      <button @click="sendMessage" :disabled="!newMessage">
        <svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="30" height="30" fill="currentColor" viewBox="0 0 24 24" aria-label="Create a post"><path d="M2,21L23,12L2,3V10L17,12L2,14V21Z"></path></svg>
      </button>
      <div v-if="showDropdown" :style="dropdownStyle" class="mention-dropdown" ref="dropdownRef">
        <div class="loader-container" v-if="loader">
          <div class="loader"></div>
        </div>
        <template v-if="users?.length">
          <div v-for="(user, index) in users" :key="index" class="mention-item" @click="handleSlct(user.username)">
            {{ user.username }}
          </div>
        </template>
        <p v-else>No Search results found</p>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { nextTick, onMounted, ref, watch } from 'vue';
import { cmpltUsers } from './SampleResponse';

  let chatMessages = ref<HTMLDivElement>()
  let msgCopy = ref<HTMLDivElement>()
  const dropdownRef = ref<HTMLDivElement>()
  const textarea = ref<HTMLTextAreaElement>();
  const chatContainer = ref<HTMLDivElement>();
  const chatInput = ref<HTMLDivElement>();

  let newMessage = ref("")
  let cursorPos = ref(0)
  let loader = ref(false)
  let isBack = ref(false)
  let isChar = ref(false)
  let showDropdown = ref(false);
  let dropdownStyle = ref<{ bottom: string, left: string }>({
    left: '0px',
    bottom: '0px'
  });
  let slctdUsers = ref<{ startPos: number, endPos: number, value: string }[]>([])

  const messages = ref<Record<string, { type: string, msg: string }[]>>({})
  const users = ref<any>([])
  users.value = JSON.parse(JSON.stringify(cmpltUsers))

  onMounted(() => {
    if (textarea.value) textarea.value.focus()
    document.addEventListener('click', hideDropdown);
  })

  watch([cursorPos,isChar,isBack], ([_newCrsrPos], [_oldCrsrPos]) => {
    // updating the selected users text position if user is changing in between texts
    if (isChar.value || isBack.value) {
      const slctUsers = slctdUsers.value.filter(f => f.startPos > (cursorPos.value - 1))
      for (let user of slctUsers) {
        user.startPos = isBack.value ? user.startPos - 1 : user.startPos + 1
        user.endPos = isBack.value ? user.endPos - 1 : user.endPos + 1
      }
    }
  });

  // disbling dropdown when click outside
  function hideDropdown(event: Event) {
    const target = event.target as HTMLElement;
    if (
      textarea.value &&
      dropdownRef.value &&
      !textarea.value.contains(target) &&
      !dropdownRef.value.contains(target)
    ) {
      showDropdown.value = false;
    }
  };

  // handler for arrow keys
  function handleKeydown(e: KeyboardEvent){
    const charKey = e.key.length === 1;
    isChar.value = charKey && e.key != 'Backspace';
    isBack.value = e.key === 'Backspace';
    if (!isChar.value && !isBack.value){
      if (textarea.value)
      cursorPos.value = textarea.value.selectionStart;
    }
  }

  // input handler
  const handleInput = (event: Event) => {
    const target = event.target as HTMLTextAreaElement;
    cursorPos.value = target.selectionStart
    const textTillCursor = newMessage.value.slice(0, cursorPos.value);
    const lttrAtCursor = textTillCursor[textTillCursor.length - 1]
    
    // handler for dropdown
    if (lttrAtCursor != '@') {
      const atPos = textTillCursor.lastIndexOf('@');
      if (atPos == -1) return showDropdown.value = false
      let fromAtToCursor = newMessage.value.slice(atPos, cursorPos.value)
      if (fromAtToCursor.indexOf("  ") != -1) return showDropdown.value = false
      let strWoAt = fromAtToCursor.substring(1)
      users.value = cmpltUsers.filter(f => f.username.toLowerCase().includes(strWoAt.trim().toLowerCase()))
      return
    }
    users.value = JSON.parse(JSON.stringify(cmpltUsers))

    // calculating the dropdowns position based on the cursor position
    if (msgCopy.value && !showDropdown.value) {
      msgCopy.value.style.width = (target.offsetWidth - 10) + 'px';
      msgCopy.value.style.font = window.getComputedStyle(target).font;
      msgCopy.value.textContent = textTillCursor;

      const marker = document.createElement('span');
      marker.textContent = '|';
      msgCopy.value.appendChild(marker);

      const markerRect = marker.getBoundingClientRect();
      if (chatContainer.value) {
        let contnrRect = chatContainer.value.getBoundingClientRect()
        let dropdownMaxPos = chatInput.value ? (chatInput.value?.offsetWidth * 0.59) : 0
        const relativeLeft = markerRect.left - contnrRect.left;
        const relativeBottom = contnrRect.bottom - markerRect.bottom;

        dropdownStyle.value.left = relativeLeft > dropdownMaxPos ? `${dropdownMaxPos}px` : `${relativeLeft}px`
        dropdownStyle.value.bottom = `${relativeBottom + 40}px`
        showDropdown.value = true
      }
    }
  };

  // async function callApi(){
  //   try {
  //     loader.value = true
  //     const respnse = await fetch("https://www.oneupapp.io/gettags", {
  //       method: 'Post',
  //       headers: {
  //         'Content-Type': 'application/json'
  //       },
  //       body: JSON.stringify({ query: newMessage.value })
  //     })
  //     console.log(respnse,'respnse')
  //   } catch (error) {
  //     console.log(error);
  //   }
  // }

  function handleSlct(name: string){
    let textTillCp = newMessage.value.slice(0, cursorPos.value);
    let prefixAtPos = textTillCp.lastIndexOf('@')
    let prefix = textTillCp.slice(0,prefixAtPos)
    let suffix = newMessage.value.slice(cursorPos.value)
    // updated mesaage along with the selected user
    newMessage.value = `${prefix}@${name}${suffix}`

    // adding selected users cuurent position
    let startPos = newMessage.value.lastIndexOf(`@${name}`)
    let endPos = (newMessage.value.lastIndexOf(name) + (name.length - 1))
    let value = `@${name}`
    slctdUsers.value.push({
      startPos,
      endPos,
      value
    })

    // updating selected users position if other users position has been changed
    const slctUsers = slctdUsers.value.filter(f => f.startPos > (startPos))
    for(let user of slctUsers){
      user.startPos = user.startPos + (value.length - 1)
      user.endPos = user.endPos + (value.length - 1)
    }
    slctdUsers.value.sort((a, b) => a.startPos - b.startPos);
    
    // updating cursor to the latt index of selected user
    if (textarea.value){
      let pos = newMessage.value.lastIndexOf(`@${name}`)
      let index = pos + `@${name}`.length
      nextTick(() => {
        if(textarea.value){
          textarea.value.setSelectionRange(index,index)
          textarea.value.focus()
        }
      })
    }
    showDropdown.value = false
  }

  function sendMessage() {
    if (newMessage.value.trim() !== '') {
      let arr = []
      let slctUsers = slctdUsers.value
      
      if (!slctUsers?.length) arr.push({ type: 'text',msg: newMessage.value })
      else{
        for(let i = 0; i < slctUsers?.length;i++){
          // adding first characters before tags or first tags
          if (!arr?.length){
            let text = newMessage.value.slice(0, slctUsers[i].startPos)
            if (text)
            arr.push({ type: 'text',msg: text })
          }
          // adding middle characters which are not tags
          if (slctUsers[i - 1]){
            if (slctUsers[i].startPos - slctUsers[i - 1].endPos > 1){
              let middle = newMessage.value.slice(slctUsers[i - 1].endPos + 1,slctUsers[i].startPos)
              arr.push({ type: 'text',msg:  middle})
            }
          }
          // matching with the tags position
          let text1 = newMessage.value.slice(slctUsers[i].startPos,slctUsers[i].endPos + 1)
          if (text1 == slctUsers[i].value) arr.push({ type: 'mention',msg: text1 })
          else arr.push({ type: 'text',msg: text1 })
          // last characters after tags
          if (!slctUsers[i + 1]){
            let last = newMessage.value.slice(slctUsers[i].endPos + 1,newMessage.value.length)
            if (last)
            arr.push({ type: 'text',msg: last })
          }
        }
      }
      let id = "ref_" + Math.random().toString(16).slice(5)
      messages.value[id] = arr
      newMessage.value = '';
      slctdUsers.value = []
      showDropdown.value = false
      cursorPos.value = 0
      nextTick(() => {
        if(chatMessages.value)
        chatMessages.value.scrollTop = chatMessages.value.scrollHeight;
      });
    }
  }
</script>


<style scoped>
  .chat-container {
    display: flex;
    flex-direction: column;
    height: 90vh;
    max-width: 60%;
    margin: 40px auto;
    border-radius: 10px;
    overflow: hidden;
    gap: 20px;
  }

  .tag{
    color: #0056b3;
  }

  .chat-messages {
    flex: 1;
    overflow-y: auto;
    padding: 20px;
    background: #edeaea;
    border-radius: 15px;
  }

  .message {
    margin-bottom: 10px;
    padding: 17px;
    background: #ffffff;
    border-radius: 5px;
    word-wrap: break-word;
    width: fit-content;
  }

  .message > span {
    white-space: pre-wrap;
  }

  .chat-input {
    display: flex;
    align-items: center;
    background: #fff;
    position: relative;
    border: 1px solid #ccc;
    border-radius: 15px;
    padding: 10px 10px;
  }

  .chat-input textarea {
    flex: 1;
    padding: 10px;
    outline: none; 
    box-shadow: none;
    /* border: 0px solid #ccc; */
    border: none;
    border-radius: 5px;
    margin-right: 10px;
    resize: none;
    height: 40px;
  }

  .chat-input textarea:hover {
    border: none;
    box-shadow: none;
  }

  .chat-input button {
    padding: 10px 20px;
    border: none;
    border-radius: 5px;
    background-color: #007bff;
    color: white;
    cursor: pointer;
    transition: background-color 0.4s ease;;
  }

  .chat-input button:disabled {
    background-color: #b1b1b1eb;
  }

  .chat-input button:not(:disabled):hover {
    background-color: #0056b3;
  }

  .mention-dropdown {
    position: absolute;
    background: white;
    border: 1px solid #ccc;
    border-radius: 5px;
    /* min-height: 135px; */
    z-index: 10;
    max-height: 150px;
    overflow-y: auto;
    width: 400px; /* Adjust width if needed */
    padding: 15px;
  }

  .mention-item {
    padding: 5px 10px;
    cursor: pointer;
  }

  .mention-item:hover {
    background-color: #f1f1f1;
  }

  .text-area-container{
    position: relative;
    flex: 1;
    display: flex;
  }

  #hidden-div{
    visibility: hidden;
    position: absolute;
    padding: 10px;
    margin-right: 10px;
    white-space: pre-wrap;
    word-wrap: break-word;
    overflow-wrap: break-word;
  }

  .loader-container {
    display: flex;
    justify-content: center;
    align-items: center;
    width: 100%;
    height: 100%;
  }

  /* HTML: <div class="loader"></div> */
  .loader {
    width: 20px;
    aspect-ratio: 1;
    border-radius: 50%;
    border: 4px solid #514b82;
    animation:
      l20-1 0.8s infinite linear alternate,
      l20-2 1.6s infinite linear;
  }
  @keyframes l20-1{
    0%    {clip-path: polygon(50% 50%,0       0,  50%   0%,  50%    0%, 50%    0%, 50%    0%, 50%    0% )}
    12.5% {clip-path: polygon(50% 50%,0       0,  50%   0%,  100%   0%, 100%   0%, 100%   0%, 100%   0% )}
    25%   {clip-path: polygon(50% 50%,0       0,  50%   0%,  100%   0%, 100% 100%, 100% 100%, 100% 100% )}
    50%   {clip-path: polygon(50% 50%,0       0,  50%   0%,  100%   0%, 100% 100%, 50%  100%, 0%   100% )}
    62.5% {clip-path: polygon(50% 50%,100%    0, 100%   0%,  100%   0%, 100% 100%, 50%  100%, 0%   100% )}
    75%   {clip-path: polygon(50% 50%,100% 100%, 100% 100%,  100% 100%, 100% 100%, 50%  100%, 0%   100% )}
    100%  {clip-path: polygon(50% 50%,50%  100%,  50% 100%,   50% 100%,  50% 100%, 50%  100%, 0%   100% )}
  }
  @keyframes l20-2{ 
    0%    {transform:scaleY(1)  rotate(0deg)}
    49.99%{transform:scaleY(1)  rotate(135deg)}
    50%   {transform:scaleY(-1) rotate(0deg)}
    100%  {transform:scaleY(-1) rotate(-135deg)}
  }

  @media (max-width: 600px) {
      .chat-container {
        max-width: 100%;
      }
  }
</style>