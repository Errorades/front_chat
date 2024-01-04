<template>
  <v-main class="pa-5">
    <h1>Chat App</h1>

    <div id="userList">
      <h2>Пользователь: {{ myID }}</h2>
    </div>

    <div id="startChat">
      <h2>Начать чат:</h2>
      <v-text-field v-model="otherUserID" label="Введите идентификатор пользователя"></v-text-field>
      <v-btn @click="startChatByID">Начать</v-btn>
    </div>

    <div>
      <h2>Чаты:</h2>
      <div>
        <ul v-for="(k, index) in Object.keys(chats)" :key="k + index" class="blue-grey lighten-4"
            @click="selectedChatID=k"
            style="cursor: pointer">
          {{ k }}
        </ul>
      </div>
    </div>

    <div v-if="selectedChatID != null">
      <h2 id="chatTitle">{{ selectedChatID }}</h2>
      <div class="message-content d-none d-md-block">
        <v-row>
          <v-col>КОМПУТЕРНЫЙ РАЗМЕР</v-col>
        </v-row>
        <v-row v-for="(message, index) in chats[selectedChatID].messages" :key="index" class="message-list">
          <v-col>
            <span
                :class="{ 'float-left':message.includes(myID + ':'), 'user-message': message.includes(myID + ':'), 'other-user-message': !message.includes(myID + ':')  }">
              {{ message }}
            </span>
          </v-col>
        </v-row>
      </div>

      <div style="width: 100%" class="message-content d-block d-md-none">
        <v-row>
          <v-col>МОБИЛЬНЫЙ РАЗМЕР</v-col>
        </v-row>
        <v-row v-for="(message, index) in chats[selectedChatID].messages" :key="index" class="message-list">
          <v-col>
            <span
                :class="{'user-message': message.includes(myID + ':'), 'other-user-message': !message.includes(myID + ':') }">
              {{ message }}
            </span>
          </v-col>
        </v-row>
      </div>
      <v-text-field v-model="msgText" label="Введите сообщение" @keydown.enter="sendMsg"></v-text-field>
      <v-btn class="ma-3" @click="sendMsg">Отправить</v-btn>
    </div>
  </v-main>
</template>

<style>
.message-list {
  overflow-y: auto; /* Включение вертикальной прокрутки для сообщений */
  max-height: 400px; /* Установка максимальной высоты списка сообщений */
  margin-bottom: 10px; /* Добавление некоторого пространства внизу списка сообщений */
}

.message-content {
//display: inline-block; max-height: 70vh; word-wrap: break-word; border-radius: 10px;
  padding: 10px;
  margin-bottom: 10px; /* Отступ снизу */
}

.other-user-message {
  display: flex;
//max-width: 70%; word-wrap: break-word; background-color: #AABFFF; /* Фоновый цвет сообщений других пользователей */ border-radius: 10px;
  padding: 10px;
  margin-bottom: 10px; /* Отступ снизу */
  float: left;
  clear: both;
}

.user-message {
  display: flex;
//max-width: 70%; word-wrap: break-word; background-color: #55AAFF; /* Фоновый цвет сообщений других пользователей */ border-radius: 10px;
  padding: 10px;
  margin-bottom: 10px; /* Отступ снизу */
  float: right;
}

#chatTitle {
  margin-top: 20px; /* Отступ сверху для заголовка чата */
}

/*@media screen and (min-width: 60vw) {
  .user-message {
    flex-basis: 100%;
    max-width: 100%;
    float: left;
  }
}*/
</style>


<script>

const forge = require('node-forge');
const crypto = import('crypto');
const rsa = forge.pki.rsa;
const io = require("socket.io-client");
global.Buffer = require('buffer').Buffer;

export default {

  name: 'ChatList',
  components: {},
  data: () => ({
    otherUserID: null,
    msgText: null,
    chats: {},
    selectedChatID: null,
    myID: null,
    socket: null,
    key: null,
    publicKey: null,
    privateKey: null,
    publicKeys: []
  }),
  created() {


    const {publicKey, privateKey} = crypto.generateKeyPairSync('ec', {
      namedCurve: 'secp256k1',    // Options
      publicKeyEncoding: {
        type: 'spki',
        format: 'pem'
      },
      privateKeyEncoding: {
        type: 'pkcs8',
        format: 'pem'
      }
    });
    this.privateKey = privateKey;
    this.publicKey = publicKey
        this.key = rsa.generateKeyPair({bits: 512, e: 0x10001});

        this.socket = io.connect("localhost:5000");

        this.socket.on("user_connected", (data) => {
          this.myID = data.id
          this.socket.emit("send_public_key", {"public_key": forge.pki.publicKeyToPem(this.key.publicKey)})
        })

        this.socket.on("chat_joined", (data) => {
          if (this.chats[data.id] === undefined) {
            let otherUser = data.users.filter((a) => a !== this.myID)[0]
            this.$set(this.chats, data.id, {"subscriber": otherUser, messages: []})
            this.socket.emit('request_public_key', {"user_id": otherUser})
          }
        })

        this.socket.on("message_sent", (data) => {

      if (this.chats[data.chat_id] !== undefined) {
        let a = this.chats[data.chat_id]
        if (data.message.user_id !== this.myID) {
          let m = this.key.privateKey.decrypt(data.message.message, 'RSA-OAEP');
          a.messages.push("[" + data.message.timestamp + "] " + data.message.user_id + ": " + forge.util.decodeUtf8(m))
          this.$set(this.chats, data.chat_id, a)
        }
      }
    })

    this.socket.on("request_public_key", (data) => {
      this.$set(this.publicKeys, data.user_id, data.public_key)
    })


  },
  methods: {
    getCurrentFormatTime() {
      const date = new Date();
      const month = date.toLocaleString('default', {month: 'long'});
      const day = date.getDate();
      const time = date.toLocaleTimeString([], {hour: '2-digit', minute: '2-digit'});
      return `${month} ${day}, ${time}`
    },
    startChatByID() {
      this.socket.emit('join_chat', {'user_id': this.otherUserID})
    },
    sendMsg() {
      if (!this.msgText || this.msgText.trim() === '') {
        // Пустое сообщение, не выполнять отправку
        return;
      }

      const currentChat = this.chats[this.selectedChatID];


      let msg = {

        'chat_id': this.selectedChatID,
        'message': forge.pki.publicKeyFromPem(
            this.publicKeys[this.chats[this.selectedChatID].subscriber]).encrypt(
            forge.util.encodeUtf8(this.msgText), 'RSA-OAEP'
        ),
        'timestamp': this.getCurrentFormatTime()
      };

      currentChat.messages.push("[" + this.getCurrentFormatTime() + "] " + this.myID + ": " + this.msgText);
      this.$set(this.chats, this.selectedChatID, currentChat);
      this.socket.emit('send_message', msg);
      this.msgText = '';

    },
  }
}
</script>
