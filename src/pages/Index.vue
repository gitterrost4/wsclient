<template>
  <div class="q-pa-md">
    <q-layout view="hHh lpR fFf">
      <q-header elevated class="bg-white black">
        <div class="order-first">
          <q-input outlined v-model="currentUrl" label="URL">
            <q-menu>
              <q-list>
                <q-item v-for="url in recentUrls" :key="url" clickable v-close-popup @click="currentUrl=url">
                  <q-item-section>
                    {{ url }} 
                  </q-item-section>
                </q-item>
              </q-list>
            </q-menu>
            <template v-slot:append>
              <q-icon name="help">
                <q-tooltip>This is the server URL you want to connect with. This usually starts with ws:// or wss://</q-tooltip>
              </q-icon>
            </template>
          </q-input>
          <q-btn @click="connect()" label="Connect" class="bg-primary"/><q-icon name="radio_button_checked" :color="stateColor"/> <span style='color:red'>{{ error }}</span> 
        </div>
      </q-header>
      <q-page-container ref="pageChat">
        <q-page class="column justify-end">
          <q-chat-message v-for="msg in messages" :key="msg.id" text-sanitize :text="[msg.message.substr(0,100)+((msg.message.length>100&&'...')||'')]" @click="showMessage(msg)" :sent="msg.sent" :stamp="msg.timestamp.toTimeString()"/>
        </q-page>
      </q-page-container>
      <q-footer class="bg-white black">
        <div class="order-last">
          <q-input clearable label="Message" autogrow input-style="max-height:40vh" outlined type="textarea" v-model="currentMessage" @keydown="keydown">
            <template v-slot:prepend>
              <q-icon name="folder" class="cursor-pointer">
                <q-menu>
                  <q-list>
                    <q-item v-for="msg in recentMessages" :key="msg" clickable v-close-popup @click="currentMessage=msg">
                      <q-item-section>
                        {{ msg.substr(0,100) }} 
                      </q-item-section>
                    </q-item>
                  </q-list>
                </q-menu>                
              </q-icon>
            </template>
            <template v-slot:append>
              <q-icon name="help">
                <q-tooltip>This is the message you want to send to the server. Hit the folder icon to choose from your last sent messages.</q-tooltip>
              </q-icon>
            </template>
          </q-input>
          <q-btn label="Send" class="bg-primary" :disable="!sendable" @click="sendMessage()"/>
        </div>
      </q-footer>
      <q-dialog v-model="showFullMessage" style="max-width:75vw" auto-close>
        <q-card style="max-width:75vw; width:75vw; max-height: 75vh">
          <q-card-actions>
            <q-btn icon="edit" label="Edit as new" @click="!!activeMessage&&!!activeMessage.message&&setCurrentMessage(activeMessage.message)"/>
          </q-card-actions>
          <q-card-section>
            <pre>{{ !!activeMessage&&!!activeMessage.message&&formatMessage(activeMessage.message) }}</pre>
          </q-card-section>
        </q-card>
      </q-dialog>
    </q-layout>
  </div>
</template>

<script lang="ts">
import { Vue, Component } from 'vue-property-decorator';
import format from 'xml-formatter'

@Component({
})
export default class PageIndex extends Vue {
  messages:Message[]=[]
  currentMessage='';
  activeMessage:Partial<Message>={}
  currentUrl='';
  websocket?:WebSocket
  stateColor='red'
  sendable=false
  error='not connected'
  showFullMessage=false;
  recentUrls:string[]=[]
  recentMessages:string[]=[]
  tabEnabled=true;

  sendMessage(){
    if(!!this.websocket){
      this.websocket.send(this.currentMessage);
      this.recentMessages.unshift(this.currentMessage)
      this.recentMessages=this.recentMessages.slice(0,10)
      localStorage.recentMessages=JSON.stringify(this.recentMessages)
      this.messages.push({id:this.messages.length,sent:true, message:this.currentMessage,timestamp:new Date()});
      this.scrollToBottom(true)
    }
  }

  mounted(){
    if(!!localStorage.recentUrls){
      this.recentUrls=JSON.parse(localStorage.recentUrls) as string[]
    }
    if(!!localStorage.recentMessages){
      this.recentMessages=JSON.parse(localStorage.recentMessages) as string[]
    }
    this.scrollToBottom(true)
  }

  recieveMessage(m:string){
    this.messages.push({id:this.messages.length,sent:false, message:m,timestamp:new Date()});
    this.scrollToBottom(true)
  }

  pushUrl(val:string){
    if(this.recentUrls.includes(val)){
      this.recentUrls = this.recentUrls.filter(x=>x!==val)
    }
    this.recentUrls.unshift(val)
    this.recentUrls=this.recentUrls.slice(0,10)
    localStorage.recentUrls=JSON.stringify(this.recentUrls);
  }

  setCurrentMessage(msg:string){
    this.currentMessage=this.formatMessage(msg)
  }

  scrollToBottom(force=false){
    setTimeout(()=>{const pageChat=(this.$refs.pageChat as Vue).$el
      if(force || window.scrollY>pageChat.scrollHeight-50) { //this is not working yet
        window.scrollTo(0,pageChat.scrollHeight)
      }
    },10)
  }

  connect(){
    this.pushUrl(this.currentUrl)
    try{
      this.websocket=new WebSocket(this.currentUrl);
      this.websocket.onmessage=m=>this.recieveMessage(m.data)
      this.stateColor='yellow'
      this.error=''
      this.websocket.onopen=()=>{this.stateColor='green'; this.sendable=true;}
      this.websocket.onerror=()=>{this.stateColor='red'; this.error='Unknown Error Occured'};
      this.websocket.onclose=(e)=>{this.stateColor='red';this.error=`${e.code}: ${e.reason}`};
    } catch(error){
      this.stateColor='red'
      this.error=error as string
    }
  }

  showMessage(msg:Message){
    this.activeMessage=msg;
    this.showFullMessage=true
  }

  formatMessage(msg:string){
    try{
      return format(msg);
    } catch(error){
      return msg;
    }
  }

  insertTab(event:{target:HTMLInputElement}){
    if(!!event&&!!event.target&&!!event.target.selectionStart){
      let originalSelectionStart = event.target.selectionStart
      let startText=this.currentMessage.slice(0,(originalSelectionStart))
      let endText=this.currentMessage.slice(originalSelectionStart)
      this.currentMessage=`${startText}\t${endText}`
      setTimeout(()=>event.target.selectionEnd=event.target.selectionStart=originalSelectionStart+1,10)
    }
  }

  //stolen from https://stackoverflow.com/questions/6637341/use-tab-to-indent-in-textarea#comment116748813_45396754
  keydown(evt: KeyboardEvent){
    const textarea = <HTMLInputElement>evt.target;
    switch(evt.key){
        case 'Escape':
            evt.preventDefault();
            this.tabEnabled = !this.tabEnabled;
            return false;
        case 'Enter':
            //break;
            if (textarea.selectionStart == textarea.selectionEnd)
            {
                // find start of the current line
                var sel = textarea.selectionStart;
                if(!!sel){
                  var text = textarea.value;
                  while (sel > 0 && text[sel-1] != '\n')
                  {
                      sel--;
                  }
                  
                  var lineStart = sel;
                  while (text[sel] == ' ' || text[sel]=='\t')
                  sel++;
                  
                  if (sel > lineStart)
                  {
                      evt.preventDefault();
                      // Insert carriage return and indented text
                      document.execCommand('insertText', false, '\n' + text.substr(lineStart, sel-lineStart));

                      // Scroll caret visible
                      textarea.blur();
                      textarea.focus();
                      return false;
                  }
                }
            }
            break;
        case 'Tab':
            if(!this.tabEnabled) break;
            evt.preventDefault();
            // selection?
            if (textarea.selectionStart == textarea.selectionEnd)
            {
                // These single character operations are undoable
                if (!evt.shiftKey)
                {
                    document.execCommand('insertText', false, '\t');
                }
                else
                {
                  if(!!textarea.selectionStart){
                    var text = textarea.value;
                    if (textarea.selectionStart > 0 && text[textarea.selectionStart-1]=='\t')
                    {
                        document.execCommand('delete');
                    }
                  }
                }
            }
            else
            {
                // Block indent/unindent trashes undo stack.
                // Select whole lines
                var selStart = textarea.selectionStart;
                var selEnd = textarea.selectionEnd;
                if(!!selStart && !!selEnd){
                  var text = textarea.value;
                  while (selStart > 0 && text[selStart-1] != '\n')
                      selStart--;
                  while (selEnd > 0 && text[selEnd-1]!='\n' && selEnd < text.length)
                      selEnd++;

                  // Get selected text
                  let lines = text.substr(selStart, selEnd - selStart).split('\n');

                  // Insert tabs
                  for (var i=0; i<lines.length; i++)
                  {
                      // Don't indent last line if cursor at start of line
                      if (i==lines.length-1 && lines[i].length==0)
                          continue;

                      // Tab or Shift+Tab?
                      if (evt.shiftKey)
                      {
                          if (lines[i].startsWith('\t'))
                              lines[i] = lines[i].substr(1);
                          else if (lines[i].startsWith('    '))
                              lines[i] = lines[i].substr(4);
                      }
                      else
                          lines[i] = '\t' + lines[i];
                  }
                  let output = lines.join('\n');

                  // Update the text area
                  textarea.value = text.substr(0, selStart) + output + text.substr(selEnd);
                  textarea.selectionStart = selStart;
                  textarea.selectionEnd = selStart + output.length; 
                }
            }
            return false;
    }
    this.tabEnabled = true;
    return true;
  }  


};

interface Message{
  id:number,sent:boolean,message:string,timestamp:Date
}
</script>
