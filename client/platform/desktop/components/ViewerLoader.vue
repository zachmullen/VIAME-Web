<script lang="ts">
import { defineComponent } from '@vue/composition-api';

import Viewer from 'viame-web-common/components/Viewer.vue';
import RunPipelineMenu from 'viame-web-common/components/RunPipelineMenu.vue';

import { getDataset } from '../store/dataset';

export default defineComponent({
  components: {
    RunPipelineMenu,
    Viewer,
  },
  props: {
    path: {
      type: String,
      required: true,
    },
  },
  setup(props) {
    const dataset = getDataset(props.path);
    return { dataset };
  },
});
</script>

<template>
  <Viewer :dataset-id="path">
    <template #title>
      <v-tabs
        icons-and-text
        hide-slider
        style="flex-basis:0; flex-grow:0;"
      >
        <v-tab to="/recent">
          Recent
          <v-icon>mdi-folder-open</v-icon>
        </v-tab>
        <v-tab to="/settings">
          Settings<v-icon>mdi-settings</v-icon>
        </v-tab>
      </v-tabs>
      <span
        v-if="dataset"
        class="title pl-3"
      >
        {{ dataset.name }}
      </span>
    </template>
    <template #title-right>
      <RunPipelineMenu :selected-dataset-ids="[path]" />
    </template>
  </Viewer>
</template>
