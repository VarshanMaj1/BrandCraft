<script setup lang="ts">
  import DashboardNavigation from '@/components/layout/DashboardNavigation.vue';
  import connectionsDataStore from '@/utils/connectionsDataStore';

  import { onMounted, ref, watch, computed } from 'vue';
  import SocialAccountCard from '@/components/views/DashboardViews/BrandsView/SocialAccountCard.vue';
  import SocialMediaInfo from '@/components/views/DashboardViews/BrandsView/SocialMediaInfo.vue';
  import { Layers } from 'lucide-vue-next';
  import { useThemeStore } from '@/utils/themeStore';

  const themeStore = useThemeStore();

  const activeCard = ref<
    | 'summary'
    | 'twitter'
    | 'threads'
    | 'bluesky'
    | 'tiktok'
    | 'instagram'
    | 'youtube'
    | 'mastodon'
    | 'facebook'
  >((localStorage.getItem('brandcraft_active_card') as any) || 'summary');

  // Watch for changes in activeCard and save to localStorage
  watch(activeCard, (newValue) => {
    localStorage.setItem('brandcraft_active_card', newValue);
  });

  onMounted(async () => {
    themeStore.initializeTheme();

    await connectionsDataStore.getAllAccounts();
  });

  // Compute color based on theme and active card
  const layerIconColor = computed(() => {
    const isDarkTheme = themeStore.currentTheme === 'dark';
    const isActive = activeCard.value === 'summary';

    // Adjust colors based on conditions
    if (isActive) {
      return isDarkTheme ? 'black' : 'white';
    } else {
      return isDarkTheme ? 'white' : 'black';
    }
  });
</script>

<template>
  <DashboardNavigation />
  <main class="h-[calc(100vh-65px)] p-6">
    <div class="mx-auto max-w-7xl">
      <div class="flex gap-8">
        <!-- Left Column: Connected Accounts -->
        <div class="shrink-0">
          <div
            class="add-icon-container flex h-[60px] w-[250px] cursor-pointer items-center justify-start gap-3 rounded-lg border p-2 shadow-sm transition-colors duration-200 dark:border-[#313131]"
            :class="[
              $attrs.class,
              {
                'bg-black text-[white] hover:bg-black/90 dark:bg-[#d9d9d9] dark:text-[black] dark:hover:bg-[#d9d9d9]/80':
                  activeCard === 'summary',
                'bg-white text-[black] hover:bg-gray-100 dark:bg-[#121212] dark:text-[white] dark:hover:bg-[#d9d9d9]/10':
                  activeCard !== 'summary',
              },
            ]"
            @click="activeCard = 'summary'"
          >
            <div class="flex h-10 w-10 items-center justify-center">
              <Layers :color="layerIconColor" />
            </div>
            <p
              :class="
                activeCard === 'summary'
                  ? 'text-[white] dark:text-[black]'
                  : 'text-[black] dark:text-[white]'
              "
              class="text-[12px] font-medium"
            >
              Summary
            </p>
          </div>
          <h2 class="mb-2 mt-8 text-base text-gray-700">Connected Accounts</h2>
          <div class="space-y-3">
            <SocialAccountCard
              networkName="Twitter"
              buttonColor="blue"
              :account="connectionsDataStore.twitterAccount.value?.[0] ?? null"
              :isConnecting="connectionsDataStore.isConnectingTwitter.value"
              @open="activeCard = 'twitter'"
              :data-active="activeCard === 'twitter'"
            />

            <SocialAccountCard
              networkName="Threads"
              buttonColor="blue"
              :account="connectionsDataStore.threadsAccount.value?.[0] ?? null"
              :isConnecting="connectionsDataStore.isConnectingThreads.value"
              @open="activeCard = 'threads'"
              :data-active="activeCard === 'threads'"
            />

            <SocialAccountCard
              networkName="Bluesky"
              buttonColor="blue"
              :account="connectionsDataStore.blueskyAccount.value?.[0] ?? null"
              :isConnecting="connectionsDataStore.isConnectingBluesky.value"
              @open="activeCard = 'bluesky'"
              :data-active="activeCard === 'bluesky'"
            />

            <SocialAccountCard
              networkName="TikTok"
              buttonColor="blue"
              :account="connectionsDataStore.tiktokAccount.value?.[0] ?? null"
              :isConnecting="connectionsDataStore.isConnectingTikTok.value"
              @open="activeCard = 'tiktok'"
              :data-active="activeCard === 'tiktok'"
            />

            <SocialAccountCard
              networkName="Instagram"
              buttonColor="blue"
              :account="
                connectionsDataStore.instagramAccount.value?.[0] ?? null
              "
              :isConnecting="connectionsDataStore.isConnectingInstagram.value"
              @open="activeCard = 'instagram'"
              :data-active="activeCard === 'instagram'"
            />

            <SocialAccountCard
              networkName="Facebook"
              buttonColor="blue"
              :account="null"
              :isConnecting="false"
              @open="activeCard = 'facebook'"
              :data-active="activeCard === 'facebook'"
            />

            <SocialAccountCard
              networkName="YouTube"
              buttonColor="blue"
              :account="connectionsDataStore.youtubeAccount.value?.[0] ?? null"
              :isConnecting="connectionsDataStore.isConnectingYoutube.value"
              @open="activeCard = 'youtube'"
              :data-active="activeCard === 'youtube'"
            />

            <SocialAccountCard
              networkName="Mastodon"
              buttonColor="blue"
              :account="connectionsDataStore.mastodonAccount.value?.[0] ?? null"
              :isConnecting="connectionsDataStore.isConnectingMastodon.value"
              @open="activeCard = 'mastodon'"
              :data-active="activeCard === 'mastodon'"
            />
          </div>
        </div>
        <div id="social-media-info" class="flex-1">
          <SocialMediaInfo v-if="activeCard" :type="activeCard" />
        </div>
      </div>
    </div>
  </main>
</template>

<style scoped>
  .fixed {
    position: fixed;
  }
</style>
