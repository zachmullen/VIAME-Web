
<script>
export default {
  props: {
    state: {
      type: String,
      require: true,
      default: 'default',
    },
  },
  data() {
    return {
      categories: [
        {
          name: 'General',
          data: [
            {
              name: 'Select', icon: 'mdi-mouse', actions: ['Left Click Mouse'], description: 'Left click a rectangle to select a detection/track',
            },
            {
              name: 'Select', icon: 'mdi-keyboard', actions: ['Up/Down Arrows'], description: 'Select Track',
            },
            {
              name: 'Zoom In/Out', icon: 'mdi-mouse', actions: ['Scrollwheel Up/Down'], description: 'use scrollwheel to zoom in and out',
            },
            {
              name: 'Zoom Area', icon: 'mdi-mouse', actions: ['Shift + Mouse Movement'], description: 'Zoom into a specific area',
            },
            {
              name: 'Reset zoom', icon: 'mdi-keyboard', actions: ['R Key'], description: 'Reset pan and zoom',
            },
          ],
        },
        {
          name: 'Editing Mode',
          data: [
            {
              name: 'New Track', icon: 'mdi-keyboard', actions: ['N Key'], description: 'Create a new Track/Detection',
            },
            {
              name: 'Edit Track', icon: 'mdi-mouse', actions: ['Right Click Mouse'], description: 'Right click a track to enter Edit Mode',
            },
            {
              name: 'Add Head/Tail', icon: 'mdi-keyboard', actions: ['H or G Key - Head', 'T or Y Key - Tail'], description: 'While a track is selected add head/tail annotations',
            },
            {
              name: 'Delete Head/Tail', icon: 'mdi-keyboard', actions: ['Q Key'], description: 'While a track is selected add head/tail annotations',
            },
          ],
        },
        {
          name: 'Selected Mode',
          data: [
            {
              name: 'First Frame', icon: 'mdi-keyboard', actions: ['Enter'], description: 'Go to first frame of selected track',
            },
            {
              name: 'Delete', icon: 'mdi-keyboard', actions: ['Delete'], description: 'Delete selected track',
            },
            {
              name: 'Edit Type', icon: 'mdi-keyboard', actions: ['Shift + Enter'], description: 'Choose/Edit track type',
            },
          ],
        },
        {
          name: 'Playback',
          data: [
            {
              name: 'Play', icon: 'mdi-keyboard', actions: ['Spacebar'], description: 'Spacebar will pause and start playback',
            },
            {
              name: 'Prev Frame', icon: 'mdi-keyboard', actions: ['F Key'], description: 'skip ahead 1 frame',
            },
            {
              name: 'Next Frame', icon: 'mdi-keyboard', actions: ['D Key'], description: 'skip back 1 frame',
            },
          ],
        },
        {
          name: 'Track Controls',
          data: [
            { name: 'Split Track', icon: 'mdi-call-split', actions: ['split @ current frame'] },
            { name: 'Set Keyframe', icon: 'mdi-star', actions: ['set keyframe true'] },
            { name: 'Interpolate', icon: 'mdi-vector-selection', actions: ['enable/disable interpolation'] },
          ],
        },
        {
          name: 'Other Controls',
          data: [
            { name: 'Edit Toggle', icon: 'mdi-pencil-box', actions: ['Toggle edit mode'] },
            { name: 'Seek', icon: 'mdi-map-marker', actions: ['Seek to detection'] },
          ],
        },
      ],
    };
  },
};
</script>
<template>
  <div class="d-flex justify-space-around flex-wrap px-2">
    <v-alert type="info" icon="mdi-vector-polygon" class="my-2">
      Polygon tool.  Click once for each point.  Right-click to close shape
    </v-alert>
    <v-card
      v-for="category in categories"
      id="helpdialog"
      :key="category.name"
      outlined
      flat
      width="100%"
      class="my-2"
    >
      <v-card-title>{{ category.name }}</v-card-title>
      <v-tooltip
        v-for="(item, index) in category.data"
        :key="item.name + index"
        color="red"
        top
      >
        <template v-slot:activator="{ on }">
          <v-row
            class="helpContextRow ma-0 align-center"
            v-on="on"
          >
            <v-col cols="4">
              {{ item.name }}
            </v-col>
            <v-col cols="1">
              <v-icon>{{ item.icon }}</v-icon>
            </v-col>
            <v-col
              col="6"
              class="pl-5"
            >
              <div
                v-for="action in item.actions"
                :key="action"
              >
                {{ action }}
              </div>
            </v-col>
          </v-row>
        </template>
        <span> {{ item.description }}</span>
      </v-tooltip>
    </v-card>
  </div>
</template>
<style lang="scss" scoped>

#helpdialog{
  font-family: monospace;
  font-size: 12px;
  .helpContextRow{
    &:hover{
      background-color: var(--v-dropzone-base);
    }
  }
}
</style>
