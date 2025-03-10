<script setup lang="ts">
  import { ref, onMounted, nextTick, watch } from 'vue';
  import { useToast } from 'primevue';
  import editorDataStore from '@/utils/editorDataStore';
  import { Loader2, Smile, Image as ImageIcon, Video } from 'lucide-vue-next';
  import { updatePostGroup } from '@/helpers/savePostGroup';
  import 'emoji-picker-element';
  import { uploadVideoToS3 } from '@/api/mediaApi';
  import { getCreatorInfo } from '@api/tiktokApi';
  import TikTokPresets from '@/components/common/forms/PostFormBase/TikTokPresets.vue';
  import InstagramPresets from '@/components/common/forms/PostFormBase/InstagramPresets.vue';
  import PreviewComponent from '@/components/common/forms/PostFormBase/PreviewComponent.vue';
  import YouTubePresets from '@/components/common/forms/PostFormBase/YouTubePresets.vue';
  import ValidationMessages from '@/components/editor/ValidationMessages.vue';
  import { uploadMedia } from '@/api/postApi';

  const toast = useToast();
  // const videoRef = ref<HTMLVideoElement | null>(null);

  const isLoading = ref(true);

  const replicatedValue = ref('');

  const status = ref<string>(
    editorDataStore.selectedPost.value?.status || 'draft'
  );

  async function handleMediaSelect(event: Event) {
    const input = event.target as HTMLInputElement;
    if (input.files) {
      const files = Array.from(input.files);
      const isVideo = input.accept.includes('video');

      // Set uploading state immediately
      editorDataStore.isUploading.value = true;

      // If it's a video, only allow one video per post
      if (isVideo) {
        if (editorDataStore.selectedPost.value.mediaPreviewUrls.length > 0) {
          toast.add({
            severity: 'warn',
            summary: 'Media limit exceeded',
            detail: 'You can only add one video per post',
            life: 3000,
          });
          editorDataStore.isUploading.value = false;
          return;
        }
        const videoFile = files[0];
        if (videoFile.size > 512 * 1024 * 1024) {
          // 512MB limit
          toast.add({
            severity: 'error',
            summary: 'File too large',
            detail: 'Video file size must be less than 512MB',
            life: 3000,
          });
          editorDataStore.isUploading.value = false;
          return;
        }

        const url = URL.createObjectURL(videoFile);
        editorDataStore.selectedPost.value.mediaPreviewUrls = [url];
        editorDataStore.selectedMedia.value = [videoFile];
        editorDataStore.currentMediaType.value = 'video';

        try {
          // Start upload to S3
          editorDataStore.uploadProgress.value = 0;
          editorDataStore.isSaving.value = true;

          await uploadVideoToS3(
            videoFile,
            (progress) => {
              editorDataStore.uploadProgress.value = progress;
            },
            editorDataStore.isSaving
          );

          // Refresh the current post data to ensure we have the latest media files
          await editorDataStore.refreshCurrentPost();

          // Clear the selected media array
          editorDataStore.selectedMedia.value = [];

          toast.add({
            severity: 'success',
            summary: 'Upload complete',
            detail: 'Video uploaded successfully',
            life: 3000,
          });
        } catch (error) {
          console.error('Failed to upload video:', error);
          toast.add({
            severity: 'error',
            summary: 'Upload failed',
            detail: 'Failed to upload video. Please try again.',
            life: 3000,
          });
        } finally {
          editorDataStore.isUploading.value = false;
          editorDataStore.isSaving.value = false;
        }
        return;
      }

      // For images, keep existing logic with 4 images max
      const totalAllowedMedia = 4;
      const currentMediaCount =
        editorDataStore.selectedPost.value.mediaPreviewUrls.length;
      const remainingSlots = totalAllowedMedia - currentMediaCount;

      if (remainingSlots <= 0) {
        toast.add({
          severity: 'warn',
          summary: 'Maximum media limit',
          detail: 'You can only add up to 4 media files per post',
          life: 3000,
        });
        editorDataStore.isUploading.value = false;
        return;
      }

      const newFiles = files.slice(0, remainingSlots);
      editorDataStore.selectedMedia.value = newFiles;
      editorDataStore.currentMediaType.value = 'image';

      try {
        editorDataStore.isSaving.value = true;
        editorDataStore.isUploading.value = true;

        // Get the post ID
        const postId = editorDataStore.selectedPost.value._id;
        const selectedMedia = editorDataStore.selectedMedia.value;

        // Upload the media files
        await uploadMedia(postId, selectedMedia);

        // Don't update local state here - just refresh the post data
        await editorDataStore.refreshCurrentPost();

        // Clear the selected media array
        editorDataStore.selectedMedia.value = [];

        toast.add({
          severity: 'success',
          summary: 'Media uploaded',
          detail: 'Media files uploaded successfully',
          life: 3000,
        });
      } catch (error) {
        console.error('Error uploading media:', error);
        toast.add({
          severity: 'error',
          summary: 'Upload failed',
          detail: 'Failed to upload media files. Please try again.',
          life: 3000,
        });
      } finally {
        editorDataStore.isUploading.value = false;
        editorDataStore.isSaving.value = false;
      }
    }
  }

  const handlePhotoUpload = () => {
    if (editorDataStore.currentMediaType.value === 'video') {
      toast.add({
        severity: 'error',
        detail: 'Remove the video first.',
        life: 3000,
      });
      return;
    }
    const input = document.createElement('input');
    input.type = 'file';
    input.accept = 'image/*';
    input.multiple = true;
    input.onchange = handleMediaSelect;
    input.click();
  };

  const handleVideoUpload = () => {
    if (editorDataStore.currentMediaType.value === 'image') {
      toast.add({
        severity: 'error',
        detail: 'Remove the images first.',
        life: 3000,
      });
      return;
    }
    const input = document.createElement('input');
    input.type = 'file';
    input.accept = 'video/*';
    input.multiple = false;
    input.onchange = handleMediaSelect;
    input.click();
  };

  const showEmojiPicker = ref(false);
  const emojiPickerRef = ref<HTMLDivElement | null>(null);
  const emojiButtonRef = ref<HTMLButtonElement | null>(null);

  // Close picker when clicking outside
  const handleClickOutside = (event: MouseEvent) => {
    if (
      emojiPickerRef.value &&
      !emojiPickerRef.value.contains(event.target as Node) &&
      emojiButtonRef.value &&
      !emojiButtonRef.value.contains(event.target as Node)
    ) {
      showEmojiPicker.value = false;
    }
  };

  const toggleEmojiPicker = async () => {
    showEmojiPicker.value = !showEmojiPicker.value;

    if (showEmojiPicker.value) {
      await nextTick(); // Wait for Vue to update the DOM

      if (!emojiPickerRef.value) return;

      emojiPickerRef.value.innerHTML = ''; // Clear any previous pickers
      const picker = document.createElement('emoji-picker');

      picker.addEventListener('emoji-click', (event: any) => {
        handleEmojiSelect(event);
      });

      emojiPickerRef.value.appendChild(picker);
    }
  };

  const handleEmojiSelect = (event: any) => {
    const emoji = event.detail.unicode; // Correctly access the selected emoji
    if (!emoji) return;

    // Insert emoji into the text content at cursor position
    const textarea = document.querySelector('textarea') as HTMLTextAreaElement;

    if (textarea) {
      editorDataStore.isUserEdit.value = true;

      const start = textarea.selectionStart;
      const end = textarea.selectionEnd;
      const text = editorDataStore.selectedPost.value?.content;

      const newContent =
        text?.substring(0, start) + emoji + text?.substring(end);
      editorDataStore.selectedPost.value!.content = newContent;

      debouncedSave();
      replicatedValue.value = newContent;

      // Move cursor after the inserted emoji
      setTimeout(() => {
        textarea.selectionStart = textarea.selectionEnd = start + emoji.length;
        textarea.focus();
      });
    }

    // Hide the picker after selecting an emoji
    // showEmojiPicker.value = false;
  };

  let saveTimeout: NodeJS.Timeout | null = null;

  // Handle all input changes
  const handleInput = (e: Event) => {
    editorDataStore.isUserEdit.value = true;
    debouncedSave();
    // Update replicated value (for automatic text-area extension)
    replicatedValue.value = (e.target as HTMLTextAreaElement).value;
  };

  // Watch for changes in selectedPost and update editorDataStore
  watch(
    () => editorDataStore.selectedPost.value,
    (newPost) => {
      if (newPost) {
        editorDataStore.selectedPost.value = newPost;

        // No need to convert back and forth since we already have the ISO string
        status.value = newPost.status || 'draft';
      }
    },
    { immediate: true }
  );

  // Watch for content changes when post is switched
  watch(
    () => editorDataStore.selectedPost.value?.content,
    (newContent) => {
      replicatedValue.value = newContent || '';
    }
  );

  async function handleSave() {
    try {
      editorDataStore.isSaving.value = true;
      await updatePostGroup();

      // Refresh the current post data after saving
      // await editorDataStore.refreshCurrentPost();

      // toast.add({
      //   severity: 'success',
      //   summary: 'Success',
      //   detail: 'Post saved successfully',
      //   life: 3000,
      // });
    } catch (error: any) {
      console.error('Save error:', error);
      toast.add({
        severity: 'error',
        summary: 'Error',
        detail: error?.response?.data?.message || 'Failed to save post',
        life: 3000,
      });
    } finally {
      editorDataStore.isSaving.value = false;
    }
  }

  // Debounced save function
  const debouncedSave = () => {
    // Clear any existing timeout
    editorDataStore.isUserEdit.value = true;
    if (saveTimeout) {
      clearTimeout(saveTimeout);
    }

    // Show saving indicator immediately when typing starts
    editorDataStore.isSaving.value = true;

    // Set new timeout
    saveTimeout = setTimeout(async () => {
      try {
        await handleSave();
      } finally {
        editorDataStore.isSaving.value = false;
      }
    }, 700); // 0.3 second delay
  };

  // Set initial button state based on post status
  onMounted(async () => {
    isLoading.value = false;
    document.addEventListener('click', handleClickOutside);

    // Initialize replicatedValue with current content
    replicatedValue.value = editorDataStore.selectedPost.value?.content || '';

    const tiktokPlatform = editorDataStore.selectedPost.value?.platforms?.find(
      (p: any) => p.startsWith('tiktok')
    );
    if (tiktokPlatform) {
      await getCreatorInfo(tiktokPlatform.split('-').slice(1).join('-'));
    }

    await nextTick(() => {
      isLoading.value = false;
    });

    // Clean up the timeout when component is unmounted
    return () => {
      if (saveTimeout) {
        clearTimeout(saveTimeout);
      }
    };
  });
</script>

<template>
  <transition name="fade" mode="out-in">
    <div
      v-if="!isLoading"
      class="transition-container flex w-full max-w-[1100px] flex-col items-start justify-start px-[30px]"
    >
      <!-- Loading Indicator -->
      <div
        :class="editorDataStore.isSaving.value ? 'opacity-100' : 'opacity-0'"
        class="saving-indicator absolute left-2 top-2 flex items-center gap-2 text-blue-500 transition-all duration-300"
      >
        <Loader2 class="h-4 w-4 animate-spin stroke-[gray]" />
      </div>

      <div class="flex w-full items-start justify-center gap-4">
        <!-- Middle Component -->
        <div
          class="scheduling-form border-greenBG mb-5 flex h-fit max-w-[600px] flex-grow rounded-[10px] bg-[white] dark:bg-[#121212]"
        >
          <div class="flex h-auto w-full flex-col gap-2 p-2">
            <div class="flex h-full w-full gap-8">
              <!-- Form Panel -->
              <div class="flex w-full flex-1 flex-col">
                <div
                  class="relative flex w-full flex-col rounded-[8px] dark:bg-[#1a1a1a]"
                >
                  <div
                    class="grow-wrap rounded-lg border border-[#ededed] bg-[#fafafa] p-2"
                    :data-replicated-value="replicatedValue"
                  >
                    <textarea
                      placeholder="Write your post here..."
                      ref="textareaRef"
                      v-model="editorDataStore.selectedPost.value.content"
                      class="w-full rounded-lg bg-[#fafafa] text-black dark:bg-[#1a1a1a]"
                      name="text"
                      id="text"
                      @input="handleInput"
                      @keydown.enter.shift.prevent
                    ></textarea>
                  </div>

                  <div class="mt-[10px] flex items-center justify-end gap-1">
                    <button
                      ref="emojiButtonRef"
                      @click="toggleEmojiPicker"
                      class="group relative flex items-center rounded-full px-1 py-1 text-sm text-gray-700"
                      :class="{
                        'bg-gray-200 dark:bg-[#404040]': showEmojiPicker,
                      }"
                    >
                      <Smile
                        class="h-5 w-5 stroke-gray-500 group-hover:stroke-black"
                      />
                    </button>
                    <button
                      @click="handlePhotoUpload"
                      class="group flex items-center rounded-full px-1 py-1 text-sm text-gray-700"
                    >
                      <ImageIcon
                        class="h-5 w-5 stroke-gray-500 group-hover:stroke-black"
                      />
                    </button>
                    <button
                      @click="handleVideoUpload"
                      class="group flex items-center rounded-full px-1 py-1 text-sm text-gray-700"
                    >
                      <Video
                        class="h-5 w-5 stroke-gray-500 group-hover:stroke-black"
                      ></Video>
                    </button>
                    <!-- <button
                      class="group flex items-center rounded-full px-1 py-1 text-sm text-gray-700"
                    >
                      <MoreHorizontal
                        class="h-5 w-5 stroke-gray-500 group-hover:stroke-black"
                      />
                    </button> -->

                    <!-- Emoji Picker -->
                    <div
                      v-if="showEmojiPicker"
                      ref="emojiPickerRef"
                      class="absolute left-1/2 top-full z-50 -translate-x-1/2 rounded-lg border shadow-lg"
                    >
                      <emoji-picker></emoji-picker>
                    </div>
                  </div>
                </div>

                <!-- Upload progress -->
                <div
                  v-if="editorDataStore.uploadProgress.value > 0"
                  class="absolute left-1/2 top-1/2 -translate-x-1/2 -translate-y-1/2 text-white"
                >
                  {{ editorDataStore.uploadProgress.value }}%
                </div>

                <div class="mb-4"></div>

                <ValidationMessages />

                <!-- Settings Panel -->
                <div class="mb-4 flex flex-col gap-1">
                  <TikTokPresets
                    v-if="
                      editorDataStore.selectedPost.value?.platforms.some(
                        (p: any) => p.startsWith('tiktok')
                      )
                    "
                    :debouncedSave="debouncedSave"
                  />
                  <InstagramPresets
                    v-if="
                      editorDataStore.selectedPost.value?.platforms.some(
                        (p: any) => p.startsWith('instagram')
                      )
                    "
                    :debouncedSave="debouncedSave"
                  />
                  <YouTubePresets
                    v-if="
                      editorDataStore.selectedPost.value?.platforms.some(
                        (p: any) => p.startsWith('youtube')
                      )
                    "
                    :debouncedSave="debouncedSave"
                  />
                </div>

                <div
                  class="preview-container overflow-hidden rounded-[10px] bg-[white] dark:bg-[#313131]"
                >
                  <PreviewComponent
                    v-if="
                      editorDataStore.selectedPost.value.mediaPreviewUrls
                        .length > 0 ||
                      editorDataStore.selectedMedia.value.length > 0
                    "
                    :media-preview-urls="
                      editorDataStore.selectedPost.value.mediaPreviewUrls
                    "
                    :current-media-type="editorDataStore.currentMediaType.value"
                    :initial-video-timestamp="
                      editorDataStore.selectedPost.value.videoTimestamp
                    "
                    :debouncedSave="debouncedSave"
                    :handleSave="handleSave"
                  />
                </div>
              </div>
            </div>
          </div>
        </div>
        <!-- <div class="divider w-[0px] self-stretch bg-layoutSoft"></div> -->
      </div>
    </div>
  </transition>
</template>

<style scoped>
  /* Left component transition */
  .scheduling-form {
    transition: all 0.5s ease-in-out;
  }

  /* Right component transition */
  .preview-container {
    overflow: hidden;
    transition: all 0.3s ease-in-out;
  }

  .grow-wrap {
    display: grid;
  }

  .grow-wrap::after {
    content: attr(data-replicated-value) ' ';
    white-space: pre-wrap;
    visibility: hidden;
  }

  .grow-wrap > textarea {
    resize: none;
    overflow: hidden;
  }

  .grow-wrap > textarea,
  .grow-wrap::after {
    font: inherit;
    grid-area: 1 / 1 / 2 / 2;
  }
</style>
