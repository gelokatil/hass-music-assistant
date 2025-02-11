<template>
  <section>
    <InfoHeader :item="itemDetails" />

    <v-tabs v-model="activeTab" show-arrows grow hide-slider>
      <v-tab
        :class="activeTab == 'tracks' ? 'active-tab' : 'inactive-tab'"
        value="tracks"
      >
        {{ $t("playlist_tracks") }}</v-tab
      >
    </v-tabs>
    <v-divider />
    <ItemsListing
      itemtype="playlisttracks"
      :parent-item="itemDetails"
      :show-providers="false"
      :show-library="false"
      :show-track-number="false"
      :load-data="loadPlaylistTracks"
      :sort-keys="['position', 'sort_name', 'sort_artist', 'sort_album']"
      v-if="activeTab == 'tracks'"
    />
  </section>
</template>

<script setup lang="ts">
import ItemsListing from "../components/ItemsListing.vue";
import { filteredItems } from "../components/ItemsListing.vue";
import InfoHeader from "../components/InfoHeader.vue";
import {
  MassEventType,
  type Playlist,
  type MassEvent,
  type MediaItemType,
} from "../plugins/api";
import { api, ProviderType } from "../plugins/api";
import { watchEffect, ref, onMounted, onBeforeUnmount } from "vue";

export interface Props {
  item_id: string;
  provider: string;
}
const props = defineProps<Props>();
const activeTab = ref("");

const itemDetails = ref<Playlist>();

const loadItemDetails = async function () {
  itemDetails.value = await api.getPlaylist(
    props.provider as ProviderType,
    props.item_id
  );
  activeTab.value = "tracks";
};

watchEffect(() => {
  // load info
  loadItemDetails();
});

onMounted(() => {
  //reload if/when item updates
  const unsub = api.subscribe_multi(
    [MassEventType.MEDIA_ITEM_ADDED, MassEventType.MEDIA_ITEM_UPDATED],
    (evt: MassEvent) => {
      // refresh info if we receive an update for this item
      const updatedItem = evt.data as MediaItemType;
      if (itemDetails.value?.uri == updatedItem.uri) {
        loadItemDetails();
      } else {
        for (const provId of updatedItem.provider_ids) {
          if (
            provId.prov_type == itemDetails.value?.provider &&
            provId.item_id == itemDetails.value?.item_id
          ) {
            loadItemDetails();
            break;
          }
        }
      }
    }
  );
  onBeforeUnmount(unsub);
});

const loadPlaylistTracks = async function (
  offset: number,
  limit: number,
  sort: string,
  search?: string,
  inLibraryOnly = true
) {
  const playlistTracks = await api.getPlaylistTracks(
    props.provider as ProviderType,
    props.item_id
  );
  return filteredItems(
    playlistTracks,
    offset,
    limit,
    sort,
    search,
    inLibraryOnly
  );
};
</script>
