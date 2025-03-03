<script setup lang="ts">
import { X } from 'lucide-vue-next';
import connectionsDataStore from '@/utils/connectionsDataStore';
import { connectFacebook } from '@/api/facebookApi';
import { disconnectConnectedAccount } from '@/api/connectedAccountsApi';

async function connect() {
  connectionsDataStore.isConnectingFacebook.value = true;
  const { authUrl } = await connectFacebook();
  window.location.href = authUrl;
}

async function disconnect(platformId: string) {
  await disconnectConnectedAccount(platformId);
  connectionsDataStore.facebookAccount.value = 
    connectionsDataStore.facebookAccount.value.filter(
      (record: any) => record.platformId !== platformId
    );
}
</script>

<template>
  <div class="w-full max-w-2xl">
    <h3 class="mb-6 text-[20px] text-gray-900">Facebook</h3>
    <h2 class="mb-6 text-2xl text-gray-900">
      Streamline your Facebook presence with automated content scheduling
    </h2>
    <div>
      <div>
        <p class="mb-4 leading-relaxed text-gray-900">
          Connect your Facebook pages to schedule and manage your content efficiently.
        </p>
        <ul class="list-disc space-y-2 pl-5 text-gray-900">
          <li>Schedule posts, photos, and videos</li>
          <li>Manage multiple Facebook pages</li>
          <li>Track engagement and reach metrics</li>
        </ul>
      </div>
      <h2
        v-if="connectionsDataStore.facebookAccount.value?.length > 0"
        class="mt-[50px] text-2xl text-gray-900"
      >
        Connected Facebook Pages
      </h2>

      <div v-if="connectionsDataStore.facebookAccount.value">
        <div
          v-for="connection in connectionsDataStore.facebookAccount.value"
          class="mt-[10px] flex w-[250px] items-center justify-start gap-3 rounded-md border border-gray-300 bg-white px-2 py-2 text-xs font-medium transition-colors duration-200 dark:border-[#313131] dark:bg-[#121212]"
        >
          <div>
            <img
              :src="connection.profileImageUrl"
              :alt="connection.username"
              class="h-[30px] w-[30px] rounded-full object-cover"
            />
          </div>
          <div class="mr-auto flex h-full flex-col justify-between">
            <div class="text-xs text-gray-500">Page</div>
            <div class="text-sm font-bold text-gray-900">
              {{ connection.username }}
            </div>
          </div>
          <div
            @click="() => disconnect(connection.platformId)"
            class="cursor-pointer rounded-md p-[3px] transition-all duration-300 hover:bg-gray-100 dark:hover:bg-[#d9d9d9]/10"
          >
            <X />
          </div>
        </div>
      </div>
    </div>
    <div class="mt-[50px]">
      <h2 class="mb-6 text-2xl text-gray-900">Connect Facebook</h2>

      <div class="space-y-6">
        <div class="rounded-lg bg-blue-50 p-6 dark:bg-gray-800">
          <div class="items-start space-y-4">
            <p class="text-sm leading-relaxed text-gray-900">
              Connect your Facebook account to start managing your pages.
            </p>
          </div>
        </div>
        <div class="mt-8 flex justify-start space-x-4">
          <button
            type="button"
            :disabled="connectionsDataStore.isConnectingFacebook.value"
            @click="connect"
            class="rounded-lg bg-[#1877F2] px-5 py-3 text-sm font-medium text-white transition hover:bg-[#0c5bce] focus:outline-none focus:ring-2 focus:ring-blue-500 disabled:cursor-not-allowed disabled:opacity-50"
          >
            <span v-if="connectionsDataStore.isConnectingFacebook.value">Connecting...</span>
            <span v-else>Connect</span>
          </button>
        </div>
      </div>
    </div>
  </div>
</template>
