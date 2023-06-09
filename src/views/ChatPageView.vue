<script setup>
import { useRoute } from "vue-router";
import { ref, onMounted } from "vue";

import { set, get, ref as REF, child, getDatabase } from "firebase/database";
import { database } from "../firebase";
const dbRef = REF(getDatabase());

import { accessId, loggedUser } from "../auth";
import { drawer, contactsDrawer, contactProfileDrawer } from "../script/index";
import SideBarView from "../components/SidebarView.vue";
import ContactsBarView from "../components/ContactsBarView.vue";

const contactChatId = ref(0);
const contact = ref({});
const unitedId = ref("");

const users = ref([]);
const messages = ref([]);

onMounted(() => {
  contactChatId.value = useRoute().params.chatId;

  get(child(dbRef, "/users"))
    .then((snapshot) => {
      if (snapshot.exists()) {
        users.value = snapshot.val();
      }
    })
    .then(() => {
      users.value.filter((user) => {
        if (user.chatId === parseInt(contactChatId.value)) {
          contact.value = user;
        }
      });
    })
    .then(() => {
      setTimeout(() => {
        unitedId.value =
          parseInt(loggedUser.value.chatId) + parseInt(contactChatId.value);

        get(child(dbRef, "/chats/" + unitedId.value + "/messages"))
          .then((snapshot) => {
            if (snapshot.exists()) {
              messages.value = snapshot.val();
            }
          })
          .then(() => {
            set(REF(database, "/chats/" + unitedId.value), {
              chatEmail: loggedUser.value.email + contact.value.email,
              chatId: `${loggedUser.value.chatId} - ${contact.value.chatId}`,
              chatName: `${loggedUser.value.username} - ${contact.value.username}`,
              messages: messages.value,
            });
          });
      }, 1000);
    });
});

const addContact = () => {
  loggedUser.value = {
    ...loggedUser.value,
    contacts: [
      ...loggedUser.value.contacts,
      {
        email: contact.value.email,
        chatId: contact.value.chatId,
        username: contact.value.username,
        avatar: contact.value.avatar,
      },
    ],
  };

  set(REF(database, "/users/" + accessId.value), loggedUser.value);
};

const deleteContact = () => {
  loggedUser.value = {
    ...loggedUser.value,
    contacts: [
      ...loggedUser.value.contacts.filter((user) => {
        if (user.chatId !== contact.value.chatId) {
          return user;
        }
      }),
    ],
  };
  set(REF(database, "/users/" + accessId.value), loggedUser.value).then(() => {
    set(REF(database, "/chats/" + unitedId.value), {
      chatEmail: loggedUser.value.email + contact.value.email,
      chatId: `${loggedUser.value.chatId} - ${contact.value.chatId}`,
      chatName: `${loggedUser.value.username} - ${contact.value.username}`,
    });
  });
};

const newMessage = ref("");

const addNewMessage = () => {
  messages.value.push(
    ...[
      {
        message: newMessage.value,
        date: new Date().getHours() + ":" + new Date().getMinutes(),
        author: loggedUser.value.chatId,
      },
    ]
  );
  set(REF(database, "/chats/" + unitedId.value + "/messages"), messages.value)
    .then(() => {
      get(child(dbRef, "/chats/" + unitedId.value + "/messages")).then(
        (snapshot) => {
          if (snapshot.exists()) {
            messages.value = snapshot.val();
          }
        }
      );
    })
    .then(() => {
      newMessage.value = "";
    });
};

const change = () => {
  get(child(dbRef, "/chats/" + unitedId.value + "/messages"))
    .then((snapshot) => {
      if (snapshot.exists()) {
        messages.value = snapshot.val();
      } else {
        messages.value = "";
      }
    })
    .then(() => {
      console.log("hoverd");
    });
};
</script>

<template>
  <div>
    <header class="header">
      <v-app-bar color="grey-darken-4" density="compact">
        <template v-slot:prepend>
          <v-app-bar-nav-icon
            v-if="loggedUser.isLogin"
            @click.stop="drawer = !drawer"
          ></v-app-bar-nav-icon>
        </template>
        <v-app-bar-title
          style="cursor: pointer"
          @click="contactProfileDrawer = !contactProfileDrawer"
        >
          <v-avatar>
            <v-img :src="contact.avatar" alt="Avatar"></v-img>
          </v-avatar>
          {{ contact.username }}
        </v-app-bar-title>

        <template v-slot:append>
          <v-btn
            icon
            v-if="loggedUser.isLogin"
            @click.stop="contactsDrawer = !contactsDrawer"
          >
            <v-icon>mdi-dots-vertical</v-icon>
          </v-btn>
        </template>
      </v-app-bar>

      <v-navigation-drawer color="grey-darken-4" v-model="drawer" temporary>
        <SideBarView />
      </v-navigation-drawer>

      <v-navigation-drawer
        color="grey-darken-4"
        v-model="contactsDrawer"
        temporary
        location="right"
      >
        <ContactsBarView />
      </v-navigation-drawer>

      <v-navigation-drawer
        color="grey-darken-4"
        v-model="contactProfileDrawer"
        temporary
        location="top"
      >
        <v-list-item class="bg-grey-darken-3 pt-5 pb-5">
          <v-avatar>
            <v-img :src="contact.avatar" alt="Avatar"></v-img>
          </v-avatar>
          <h4 class="mt-4">{{ contact.username }}</h4>
          <h5 class="">{{ contact.email }}</h5>
          <h5></h5>
        </v-list-item>

        <v-list density="compact" nav>
          <v-list-item
            @click="addContact"
            prepend-icon="mdi-magnify"
            title="Add To Your Contacts"
            value=""
            class="bg-green-darken-4"
          ></v-list-item>

          <v-list-item
            @click="deleteContact"
            prepend-icon="mdi-door"
            title="Delete This Contact"
            value=""
            class="bg-red-darken-4"
          ></v-list-item>
        </v-list>
      </v-navigation-drawer>
    </header>

    <div class="mx-9">
      <v-list
        style="max-height: 500px"
        class="overflow-y-auto mt-15"
        @touchstart="change"
        @change="(messages) => change()"
        @click="change"
        @mouseenter="change"
        @mouseleave="change"
      >
        <template v-slot>
          <v-list-item
            v-for="message in messages"
            :key="message"
            min-height="100"
            :class="
              message.author === loggedUser.chatId
                ? 'bg-indigo-darken-3 mt-3 mx-auto me-5 rounded-xl'
                : 'bg-grey-darken-3 mt-3 rounded-xl'
            "
            width="60%"
          >
            <v-list-tile>
              <v-list-tile-content>
                <br />
                {{ message.message }}
                <br /><br />
              </v-list-tile-content>
              <v-list-tile-footer> {{ message.date }} </v-list-tile-footer>
            </v-list-tile>
          </v-list-item>
        </template>
      </v-list>

      <br /><br />
      <v-text-field
        v-model="newMessage"
        density="compact"
        variant="solo"
        label="Send Message"
        append-inner-icon="mdi-send"
        single-line
        hide-details
        @click:append-inner="addNewMessage"
        @keyup.enter="addNewMessage"
      ></v-text-field>
    </div>
  </div>
</template>

<style>
.chatPosts {
  overflow-y: scroll;
}
.chatPost {
  width: 60%;
  min-height: 150px;
  display: grid;
  justify-content: center;
  align-items: center;
  margin: 0 auto;
}
.chatPost p {
  margin: 15px;
}
</style>
