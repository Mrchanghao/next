<script setup lang="ts">
import { ref, computed, watch } from "vue";
import { Howl, Howler } from "howler";

import ShuffleIcon from "@/assets/icons/ShuffleIcon.vue";
import RepeatIcon from "@/assets/icons/RepeatIcon.vue";
import BackwardIcon from "@/assets/icons/BackwardIcon.vue";
import ForwardIcon from "@/assets/icons/ForwardIcon.vue";
import PlayIcon from "@/assets/icons/PlayIcon.vue";
import PauseIcon from "@/assets/icons/PauseIcon.vue";
import HeartIcon from "@/assets/icons/HeartIcon.vue";
import VolumeIcon from "@/assets/icons/VolumeIcon.vue";
import MuteIcon from "@/assets/icons/MuteIcon.vue";

import { usePlayerStore } from "@/stores/player";
import { useUserStore } from "@/stores/user";

const playerStore = usePlayerStore();
const userStore = useUserStore();

const isPaused = ref(true);
const track = ref(0);
const isMouseDownOnTrack = ref(false);
const seekMinutes = ref("00");
const seekSeconds = ref("00");

let sound = new Howl({
  src: [`/audio/${playerStore.currnetSong.src}`],
  html5: true,
  loop: playerStore.loop,
  onload: function () {
    track.value = 0;
  },
  onplay: function () {
    isPaused.value = false;
    requestAnimationFrame(step);
  },
  onseek: function () {
    requestAnimationFrame(step);
  },
  onend: function () {
    if (!playerStore.loop) {
      if (playerStore.currentIndex === playerStore.lastIndex) {
        play();
      } else {
        playerStore.next();
      }
    }
  },
});

const durationMinutes = computed(() => {
  return zeroPad(playerStore.currnetSong.duration / 60);
});

const durationSeconds = computed(() => {
  return zeroPad(playerStore.currnetSong.duration % 60);
});

const isLiked = computed(() => {
  return userStore.isInLikes(playerStore.currnetSong.id);
});

function zeroPad(input: number) {
  return ("0" + Math.floor(input)).slice(-2);
}

function likeSong() {
  if (isLiked.value) {
    userStore.removeFromLikes(playerStore.currnetSong.id);
  } else {
    userStore.addToLikes(playerStore.currnetSong);
  }
}

function play() {
  isPaused.value = !isPaused.value;
  if (isPaused.value) {
    sound.pause();
  } else {
    sound.play();
  }
}

function mute() {
  playerStore.isMute = !playerStore.isMute;
  if (playerStore.isMute) {
    Howler.mute(true);
  } else {
    Howler.mute(false);
  }
}

function loop() {
  playerStore.loop = !playerStore.loop;
  sound.loop(playerStore.loop);
}

function step() {
  if (sound.playing()) {
    track.value = Math.round(sound.seek());
    seekMinutes.value = zeroPad(track.value / 60);
    seekSeconds.value = zeroPad(track.value % 60);
    requestAnimationFrame(step);
  }
}

watch(
  () => playerStore.currnetSong,
  (newSong) => {
    sound.unload();
    sound = new Howl({
      src: [`/audio/${newSong.src}`],
      html5: true,
      autoplay: true,
      loop: playerStore.loop,
      onload: function () {
        track.value = 0;
      },
      onplay: function () {
        isPaused.value = false;
        requestAnimationFrame(step);
      },
      onseek: function () {
        requestAnimationFrame(step);
      },
      onend: function () {
        if (!playerStore.loop) {
          if (playerStore.currentIndex === playerStore.lastIndex) {
            play();
          } else {
            playerStore.next();
          }
        }
      },
    });

    userStore.addToRecents(newSong);
  }
);

watch(track, (newTrack) => {
  if (isMouseDownOnTrack.value) {
    sound.seek(newTrack);
  }
});

watch(
  () => playerStore.volume,
  (newVolume) => {
    Howler.volume(newVolume / 10);
  }
);

watch(
  playerStore.$state,
  () => {
    playerStore.updatePlayerSettings();
  },
  { deep: true }
);
</script>

<template>
  <div
    class="fixed left-2 right-2 bottom-4 z-20 flex items-center rounded-xl bg-secondary-900/20 px-3 py-3 font-Quicksand text-secondary-900 shadow-lg backdrop-blur-3xl backdrop-brightness-100 transition-colors duration-300 dark:bg-primary-900/10 dark:text-primary-900 lg:right-4 lg:left-56 lg:justify-between lg:px-6"
  >
    <div class="flex grow items-center lg:w-3/12 lg:grow-0">
      <img
        class="h-12 w-12 rounded-xl lg:h-16 lg:w-16"
        :src="`/img/${playerStore.currnetSong.thumbnail}`"
        :alt="playerStore.currnetSong.title"
      />

      <div class="ml-3 overflow-hidden text-ellipsis whitespace-nowrap lg:ml-6">
        <router-link
          class="font-medium lg:text-lg"
          :to="'/album/' + playerStore.currnetSong.albumId"
        >
          {{ playerStore.currnetSong.title }}
        </router-link>
        <div
          class="flex text-xs font-light text-secondary-500 transition-colors duration-300 dark:text-primary-500 lg:text-sm"
        >
          <template
            v-for="(artist, index) in playerStore.currnetSong.artists"
            :key="index"
          >
            <router-link
              class="underline-offset-2 hover:underline"
              :to="'/artist/' + artist.id"
            >
              {{ artist.name }}
            </router-link>
            <span
              v-if="
                playerStore.currnetSong.artists.length !== 1 &&
                index !== playerStore.currnetSong.artists.length - 1
              "
              class="mx-1"
            >
              ,
            </span>
          </template>
        </div>
      </div>
    </div>

    <div class="lg:w-6/12">
      <div class="flex items-center justify-center lg:mb-1">
        <button
          class="mx-1 hidden h-10 w-10 items-center justify-center rounded-full outline-none transition duration-300 focus:bg-secondary-900/10 dark:focus:bg-primary-900/10 lg:flex"
          :class="[
            playerStore.shuffle
              ? 'fill-secondary-900 hover:bg-secondary-900/10  dark:fill-primary-900 dark:hover:bg-primary-900/10 '
              : 'fill-secondary-900/30 hover:fill-secondary-900/60 dark:fill-primary-900/30 dark:hover:fill-primary-900/60',
          ]"
        >
          <ShuffleIcon class="w-4" />
        </button>

        <button
          class="mx-1 hidden h-10 w-10 items-center justify-center rounded-full fill-secondary-900 outline-none transition-colors duration-300 hover:bg-secondary-900/10 focus:bg-secondary-900/10 dark:fill-primary-900 dark:hover:bg-primary-900/10 dark:focus:bg-primary-900/10 lg:flex lg:h-11 lg:w-11"
          @click="playerStore.pervious"
        >
          <BackwardIcon class="w-4" />
        </button>

        <button
          class="flex h-10 w-10 items-center justify-center rounded-full fill-secondary-900 outline-none transition-colors duration-300 hover:bg-secondary-900/10 focus:bg-secondary-900/10 dark:fill-primary-900 dark:hover:bg-primary-900/10 dark:focus:bg-primary-900/10 lg:mx-1 lg:h-12 lg:w-12"
          @click="play"
        >
          <PlayIcon v-if="isPaused" class="ml-1 w-4 lg:w-6" />
          <PauseIcon v-else class="w-4 lg:w-6" />
        </button>

        <button
          class="flex h-10 w-10 items-center justify-center rounded-full fill-secondary-900 outline-none transition-colors duration-300 hover:bg-secondary-900/10 focus:bg-secondary-900/10 dark:fill-primary-900 dark:hover:bg-primary-900/10 dark:focus:bg-primary-900/10 lg:mx-1"
          @click="playerStore.next"
        >
          <ForwardIcon class="w-3 lg:w-4" />
        </button>

        <button
          class="mx-1 hidden h-10 w-10 items-center justify-center rounded-full outline-none transition duration-300 focus:bg-secondary-900/10 dark:focus:bg-primary-900/10 lg:flex"
          :class="[
            playerStore.loop
              ? 'fill-secondary-900 hover:bg-secondary-900/10  dark:fill-primary-900 dark:hover:bg-primary-900/10 '
              : 'fill-secondary-900/30 hover:fill-secondary-900/60 dark:fill-primary-900/30 dark:hover:fill-primary-900/60',
          ]"
          @click="loop"
        >
          <RepeatIcon class="w-4" />
        </button>
      </div>

      <div class="flex items-center justify-center">
        <div class="hidden w-10 lg:block">
          {{ `${seekMinutes}:${seekSeconds}` }}
        </div>
        <div class="slide mx-4 w-[30rem]">
          <input
            class="outline-none"
            v-model="track"
            type="range"
            min="0"
            :max="playerStore.currnetSong.duration"
            @mousedown="isMouseDownOnTrack = true"
            @mouseup="isMouseDownOnTrack = false"
          />
          <progress
            :value="track"
            min="0"
            :max="playerStore.currnetSong.duration"
          ></progress>
        </div>
        <div class="hidden w-10 lg:block">
          {{ `${durationMinutes}:${durationSeconds}` }}
        </div>
      </div>
    </div>

    <div class="flex items-center justify-end lg:w-3/12">
      <button
        class="flex h-10 w-10 items-center justify-center rounded-full outline-none transition-colors duration-300 hover:bg-secondary-900/10 focus:bg-secondary-900/10 dark:hover:bg-primary-900/10 dark:focus:bg-primary-900/10 lg:mr-1 lg:h-14 lg:w-14"
        :class="[
          isLiked ? 'fill-red-500' : 'fill-secondary-900 dark:fill-primary-900',
        ]"
        @click="likeSong"
      >
        <HeartIcon class="w-5 lg:w-7" />
      </button>

      <button
        class="mr-1 hidden h-14 w-14 items-center justify-center rounded-full fill-secondary-900 outline-none transition-colors duration-300 hover:bg-secondary-900/10 focus:bg-secondary-900/10 dark:fill-primary-900 dark:hover:bg-primary-900/10 dark:focus:bg-primary-900/10 lg:flex"
        @click="mute"
      >
        <MuteIcon v-if="playerStore.isMute" class="w-7" />
        <VolumeIcon v-else class="w-7" />
      </button>

      <div class="slide w-28">
        <input
          class="outline-none"
          v-model="playerStore.volume"
          type="range"
          min="0"
          max="10"
        />
        <progress :value="playerStore.volume" min="0" max="10"></progress>
      </div>
    </div>
  </div>
</template>
