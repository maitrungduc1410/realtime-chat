<script setup>
import ListUser from "../components/ListUser.vue";
import {
  ref,
  onBeforeMount,
  onBeforeUnmount,
  inject,
  computed,
  watch,
} from "vue";
import { useRoute } from "vue-router";
import Chat from "../components/Chat.vue";
import axios from "axios";

const route = useRoute();
const currentRoom = ref({});
const selectedUser = ref(null);
const usersOnline = ref([]);
const privateRoomId = ref(null);
const rooms = inject("$rooms");
const user = inject("$user");
const appName = inject("$appName");
const showToast = inject("$showToast");

onBeforeMount(() => {
  const index = rooms.findIndex(
    (item) => item.id === parseInt(route.params.roomId)
  );
  if (index > -1) {
    currentRoom.value = rooms[index];
    Echo
      .join(`room.${currentRoom.value.id}`) // listen to the shared room
      .here((users) => {
        usersOnline.value = users;
      })
      .joining((user) => {
        usersOnline.value.push(user);

        if (selectedUser.value && user.id === selectedUser.value.id) {
          selectedUser.value.isOnline = true;
        }
      })
      .leaving((user) => {
        const index = usersOnline.value.findIndex(
          (item) => item.id === user.id
        );
        if (index > -1) {
          usersOnline.value.splice(index, 1);
        }

        if (selectedUser.value && user.id === selectedUser.value.id) {
          selectedUser.value.isOnline = false;
        }
      });

    // listen to user's own room (in order to receive all private messages from other users)
    // Echo.private(`room.${user.id}`).listen("MessagePosted", (e) => {
    //   if (!selectedUser.value) {
    //     const index = usersOnline.value.findIndex(
    //       (item) => item.id === e.message.user.id
    //     );
    //     if (index > -1) {
    //       usersOnline.value[index].new_messages++;
    //     }
    //   }
    // });
  }
});

onBeforeUnmount(() => {
  // Echo.leave(`room.${user.id}`);
  Echo.leave(`room.${currentRoom.value.id}`);
});

async function selectReceiver(receiver) {
  if (receiver.id === user.id) {
    showToast("Error", "You can't chat with yourself");
    return;
  }

  if (selectedUser.value) {
    showToast("Error", "You can only chat with one user at a time");
    return
  }

  try {
    const response = await axios.post(`/start_chat`, {
      receiver_id: receiver.id,
    });

    privateRoomId.value = response.data.id

    selectedUser.value = {
      ...receiver,
      isOnline: usersOnline.value.find(item => item.id === receiver.id),
    };

    // const user = usersOnline.value.find((item) => item.id === receiver.id);
    // if (user) {
    //   user.new_messages = 0;
    // }
  } catch (error) {
    console.error(error);
  }
}

function closeChat() {
  selectedUser.value = null;
  privateRoomId.value = null;
}

// const totalUnreadPrivateMessages = computed(() => {
//   let count = 0;
//   usersOnline.value.forEach((item) => {
//     count += item.new_messages;
//   });
//   return count;
// });

// watch(totalUnreadPrivateMessages, () => {
//   if (totalUnreadPrivateMessages.value > 0) {
//     document.title = `${totalUnreadPrivateMessages.value > 0
//         ? "(" + totalUnreadPrivateMessages.value + ")"
//         : ""
//       } - ${appName}`;
//   } else {
//     document.title = appName;
//   }
// });
</script>

<template>
  <div class="flex h-100">
    <div class="row justify-content-center h-100">
      <div class="col-md-8 chat">
        <Chat :roomId="currentRoom.id" :roomName="currentRoom.name" :roomDescription="currentRoom.description"
          @selectReceiver="selectReceiver" />
      </div>
      <div class="col-md-4 chat">
        <ListUser :usersOnline="usersOnline" @selectReceiver="selectReceiver" />
      </div>
    </div>

    <!-- private chat -->
    <Chat v-if="selectedUser && privateRoomId" isPrivate :roomId="privateRoomId" :receiver="selectedUser"
      @closeChat="closeChat" />
  </div>
</template>
