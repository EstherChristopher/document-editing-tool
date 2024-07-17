<template>
  <div class="container">
    <div class="sidebar">
      <div>
        <h3>Current User</h3>
        <div>
          <input type="text" v-model="username" :disabled="loggedIn" />
        </div>
        <button @click="login" v-if="!loggedIn">Log in as an editor</button>
      </div>

      <div>
        <!-- Display all active users editing the document -->
        <h3>Active Users</h3>
        <ul id="active-users-list">
          <li v-for="user in activeUsers" :key="user">
            {{ user }}
          </li>
        </ul>
      </div>

      <div>
        <h3>Document History</h3>
        <ul id="history-list">
          <li v-for="log in history" :key="log">
            {{ log }}
          </li>
        </ul>
      </div>
    </div>
    <div class="content">
      <div class="document-info">
        <h3>{{ document.Title }}</h3>
        <!-- <p>Created by: User Name (date)</p> -->
      </div>
      <div id="editor" style="position: relative;">
        <p v-if="!loggedIn">Log in to make edits to this document</p>
        <textarea
          class="textarea"
          ref="textareaRef"
          :disabled="!loggedIn"
          @click="getCoordinates"
          @input="updateDocument"
          v-model="document.Content"
        >
        </textarea>
      </div>

      <div
          class="cursor"
          v-for="cursor in cursorTrackers"
          :key="cursor.user"
          :clientX="cursor.x"
          :clientY="cursor.y"
          :user="cursor.user"
          :style="{
            position: 'absolute',
            left: (cursor.x) + 'px',
            top: (cursor.y) + 'px',
            color: cursor.color,
            display: loggedIn ? 'inline-block' : 'none',
            transform: 'rotate(-90deg)',
          }"
        >
          <!-- <span style="transform: rotate(-90deg)"> -->
            <i class="uil uil-location-arrow" style="font-size: 15px;"></i>
          <!-- </span> -->
          
          <span class="cursor-text" style="transform: rotate(90deg);">{{ cursor.user }}</span>
        </div>
    </div>
  </div>
</template>

<script>
import axios from "../services/strapi.js";
import io from "socket.io-client";

export default {
  beforeMount() {

    this.fetchDocument();
    
    this.socket = io("http://localhost:1337");
    this.socket.on("document-updated", () => {
      this.fetchDocument();
      console.log("Document updated");
    });

    this.socket.on("active-users", (data) => {
      this.activeUsers = data;
    });

    this.socket.on("document-updated", ({ message, updateData }) => {
      // only keeps track of the 10 latest actions
      if (this.history.length >= 10) {
        this.history.pop();
      }
      this.history.unshift(message);

      this.document = updateData;
    });

    this.socket.on("cursor-moved", ({ user, x, y }) => {
      let index = this.cursorTrackers.findIndex(
        (cursor) => cursor.user === user
      );
      if (index == -1) {
        this.cursorTrackers.push({
          user,
          x,
          y,
          color: this.generateRandomColor(),
        });
      } else {
        this.cursorTrackers[index] = {
          user,
          x,
          y,
          color: this.cursorTrackers[index].color,
        };
      }
    });
  },
  data() {
    return {
      socket: null,
      id: null,
      document: {
        Title: "",
        Content: "",
      },
      // new properties to account for the current user, active users and history.
      activeUsers: [],
      history: [],
      loggedIn: false,
      username: "",
      cursorTrackers: [],
    };
  },
  methods: {
    // the login function emits the user-joined event to the websocket
    login() {
      this.socket.emit("user-joined", this.username);
      this.loggedIn = true;
    },
    fetchDocument() {
      axios
        .get("/documents/1")
        .then(({ data }) => {
          this.id = data.data.id;
          this.document = data.data.attributes;
        })
        .catch((err) => console.log(err));
    },
    updateDocument() {
      this.socket.emit("edit-document", { id: this.id, ...this.document });
    },
    getCoordinates(event) {
      if (this.loggedIn) {
        const x = event.clientX;
        const y = event.clientY;
        
        this.socket.emit("move-cursor", { x, y, pos: event.target.selectionStart });
      }
    },
    generateRandomColor() {
      // Create a hexadecimal color string with a "#" prefix
      const color = "#";

      // Generate random numbers for red, green, and blue components (0-255)
      const red = Math.floor(Math.random() * 256)
        .toString(16)
        .padStart(2, "0");
      const green = Math.floor(Math.random() * 256)
        .toString(16)
        .padStart(2, "0");
      const blue = Math.floor(Math.random() * 256)
        .toString(16)
        .padStart(2, "0");

      // Combine the components into the final color string
      return color + red + green + blue;
    },
  },
  mounted() {
    this.fetchDocument();
  },
};
</script>

<style>
body {
  font-family: sans-serif;
  margin: 0;
  padding: 0;
  display: flex;
  min-height: 100vh;
}

.container {
  display: flex;
  flex: 1;
  min-height: 100vh;
}

.sidebar {
  width: 40%;
  padding: 20px;
  border-right: 1px solid #ddd;
  background-color: #f5f5f5;
  min-height: 100vh;
}

.sidebar h2 {
  margin-top: 0;
  margin-bottom: 10px;
}

.sidebar ul {
  list-style: none;
  padding: 0;
}

.sidebar li {
  margin-bottom: 5px;
}

.content {
  flex: 1;
  padding: 20px;
  height: 100%;
}

.document-info {
  margin-bottom: 20px;
}

.document-info h3 {
  font-size: 18px;
  margin-bottom: 5px;
}

#editor .textarea {
  width: 60vw;
  height: calc(100vh - 160px); /* Account for header and padding */
  border: 1px solid #ddd;
  padding: 10px;
  font-size: 16px;
}

input {
  padding: 8px;
  border-radius: 3px;
}

button {
  margin-top: 10px;
  padding: 6px;
  border: none;
  outline: none;
  background-color: #0d6aad;
  color: white;
  border-radius: 5px;
}

.cursor {
  display: inline-block;
  font-size: 20px;
  animation: blink 0.5s step-end infinite;
}

.cursor span {
  font-size: 12px;
  position: absolute;
  bottom: 0;
  visibility: hidden;
}

.cursor:hover span {
  visibility: visible;
}

@keyframes blink {
  0% {
    visibility: hidden;
  }
  50% {
    visibility: visible;
  }
}
</style>
