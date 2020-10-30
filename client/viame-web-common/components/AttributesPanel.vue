<script lang="ts">
import {
  computed, defineComponent, onBeforeMount, ref,
} from '@vue/composition-api';

import {
  useSelectedTrackId, useFrame, useTrackMap, useEditingMode, useTypeStyling, useAllTypes,
} from 'vue-media-annotator/provides';
import Track, { TrackId, Feature } from 'vue-media-annotator/track';
import TrackItem from 'vue-media-annotator/components/TrackItem.vue';

import { useApi, Attribute } from 'viame-web-common/apispec';
import AttributeInput from 'viame-web-common/components/AttributeInput.vue';

function getTrack(trackMap: Readonly<Map<TrackId, Track>>, trackId: TrackId): Track {
  const track = trackMap.get(trackId);
  if (track === undefined) {
    throw new Error(`Track ${trackId} missing from map`);
  }
  return track;
}

export default defineComponent({
  components: {
    AttributeInput,
    TrackItem,
  },
  props: {
    lockTypes: {
      type: Boolean,
      default: false,
    },
  },
  setup(props) {
    const attributes = ref([] as Attribute[]);
    const trackMap = useTrackMap();
    const editingModeRef = useEditingMode();
    const typeStylingRef = useTypeStyling();
    const allTypesRef = useAllTypes();

    const frameRef = useFrame();
    const selectedTrackIdRef = useSelectedTrackId();
    const { getAttributes } = useApi();
    const trackAttributes = computed(() => attributes.value.filter(
      (a) => a.belongs === 'track',
    ));
    const detectionAttributes = computed(() => attributes.value.filter(
      (a) => a.belongs === 'detection',
    ));
    const selectedTrack = computed(() => {
      if (selectedTrackIdRef.value !== null) {
        console.log(getTrack(trackMap, selectedTrackIdRef.value));
        return getTrack(trackMap, selectedTrackIdRef.value);
      }
      return null;
    });
    const trackType = computed(() => {
      if (selectedTrack.value !== null) {
        return selectedTrack.value.confidencePairs[0][0];
      }
      return '';
    });
    const selectedDetection = computed(() => {
      const t = selectedTrack.value;
      if (t !== null) {
        const [real] = t.getFeature(frameRef.value);
        return real;
      }
      return null;
    });

    function updateTrackAttribute(
      trackId: TrackId | null,
      { name, value }: { name: string; value: unknown },
    ) {
      console.log(`Updating Track Attribute: ${name} with ${value}`);
      if (trackId === null) return;
      const track = getTrack(trackMap, trackId);
      track.setAttribute(name, value);
    }

    function updateFeatureAttribute(
      trackId: TrackId | null,
      oldFeature: Feature | null,
      { name, value }: { name: string; value: unknown },
    ) {
      if (trackId === null) return;
      if (oldFeature === null) return;
      const track = getTrack(trackMap, trackId);
      track.setFeatureAttribute(oldFeature.frame, name, value);
    }

    function trackAttributeAdd() {
      console.log('Add Track Attribute');
    }
    function detectionAttributeAdd() {
      console.log('Add Track Attribute');
    }

    onBeforeMount(async () => {
      attributes.value = await getAttributes();
    });

    return {
      selectedTrackIdRef,
      /* Attributes */
      detectionAttributes,
      trackAttributes,
      /* Selected */
      selectedDetection,
      selectedTrack,
      /* Update functions */
      updateFeatureAttribute,
      updateTrackAttribute,
      trackAttributeAdd,
      detectionAttributeAdd,
      editingModeRef,
      trackType,
      typeStylingRef,
      allTypesRef,
    };
  },
});
</script>

<template>
  <div>
    <v-subheader>Track Editor</v-subheader>
    <v-container class="pa-0">
      <v-row
        class="attributes-panel pa-0 ma-0"
        dense
      >
        <v-col
          style="overflow-y: auto;"
        >
          <div
            v-if="!selectedTrack"
            class="ml-4 body-2"
          >
            No track selected
          </div>
          <template v-else>
            <v-row
              class="mx-4 body-2"
              style="line-height:22px;"
            >
              <datalist id="allTypesOptions">
                <option
                  v-for="type in allTypesRef"
                  :key="type"
                  :value="type"
                >
                  {{ type }}
                </option>
              </datalist>

              <track-item
                :single-display="true"
                :track="selectedTrack"
                :track-type="trackType"
                :selected="true"
                :editing="!!editingModeRef"
                :input-value="true"
                :color="typeStylingRef.color(trackType)"
                :lock-types="lockTypes"
                @seek="$emit('track-seek', $event)"
              />
            </v-row>
            <v-subheader class="border-highlight">
              Confidence Pairs
            </v-subheader>
            <v-row
              dense
              style="max-height:20%"
            >
              <v-col dense>
                <v-row
                  v-for="(pair, index) in selectedTrack.confidencePairs"
                  :key="index"
                  class="ml-1"
                  dense
                  style="font-size:.8em"
                >
                  <v-col>
                    {{ pair[0] }}
                  </v-col>
                  <v-col>
                    {{ pair[1].toFixed(2) }}
                  </v-col>
                </v-row>
              </v-col>
            </v-row>
            <v-subheader class="border-highlight">
              <v-row>
                Track Attributes:
                <v-spacer />
                <v-tooltip
                  open-delay="200"
                  bottom
                  max-width="200"
                >
                  <template #activator="{ on }">
                    <v-btn
                      outlined
                      x-small
                      class="mr-2"
                      v-on="on"
                      @click="trackAttributeAdd"
                    >
                      <v-icon small>
                        mdi-plus
                      </v-icon>
                      Attribute
                    </v-btn>
                  </template>
                  <span>Add a new Track Attribute</span>
                </v-tooltip>
              </v-row>
            </v-subheader>
            <v-row
              class="ma-0"
              style="max-height:20%; overflow-y:auto; overflow-x:hidden"
              dense
            >
              <v-col class="pa-2">
                <v-row
                  v-for="(attribute, i) of trackAttributes"
                  :key="i"
                  class="ma-0"
                  dense
                  align="center"
                >
                  <v-col style="font-size:.8em">
                    {{ attribute.name }}:
                  </v-col>
                  <v-col class="px-1">
                    <AttributeInput
                      :datatype="attribute.datatype"
                      :name="attribute.name"
                      :values="attribute.values ? attribute.values : null"
                      :value="
                        selectedTrack
                          ? selectedTrack.attributes[attribute.name]
                          : undefined
                      "
                      @change="updateTrackAttribute(selectedTrackIdRef, $event)"
                    />
                  </v-col>
                  <v-col cols="1">
                    <v-btn
                      icon
                      small
                    >
                      <v-icon
                        small
                      >
                        mdi-settings
                      </v-icon>
                    </v-btn>
                  </v-col>
                </v-row>
              </v-col>
            </v-row>
            <v-subheader
              v-if="selectedDetection"
              class="border-highlight"
            >
              <v-row class="pt-2">
                Detection Attributes:
                <v-spacer />
                <v-tooltip
                  open-delay="200"
                  bottom
                  max-width="200"
                >
                  <template #activator="{ on }">
                    <v-btn
                      outlined
                      x-small
                      class="mr-2"
                      v-on="on"
                      @click="detectionAttributeAdd"
                    >
                      <v-icon small>
                        mdi-plus
                      </v-icon>
                      Attribute
                    </v-btn>
                  </template>
                  <span>Add a new Detection Attribute</span>
                </v-tooltip>
                <div>
                  {{ `Frame: ${selectedDetection.frame}` }}
                </div>
              </v-row>
            </v-subheader>
            <v-subheader
              v-else
              class="border-highlight"
            >
              No detection selected
            </v-subheader>
            <v-row
              v-if="selectedDetection"
              class="ma-0"
              style="max-height:20%; overflow-y:auto; overflow-x:hidden"
              dense
            >
              <v-col class="pa-2">
                <v-row
                  v-for="(attribute, i) of detectionAttributes"
                  :key="i"
                  class="ma-0"
                  dense
                  align="center"
                >
                  <v-col style="font-size:.8em">
                    {{ attribute.name }}:
                  </v-col>
                  <v-col class="px-1">
                    <AttributeInput
                      :datatype="attribute.datatype"
                      :name="attribute.name"
                      :values="attribute.values ? attribute.values : null"
                      :value="
                        selectedDetection && selectedDetection.attributes
                          ? selectedDetection.attributes[attribute.name]
                          : undefined
                      "
                      @change="updateFeatureAttribute(
                        selectedTrackIdRef, selectedDetection, $event)"
                    />
                  </v-col>
                  <v-col cols="1">
                    <v-btn
                      icon
                      small
                    >
                      <v-icon
                        small
                      >
                        mdi-settings
                      </v-icon>
                    </v-btn>
                  </v-col>
                </v-row>
              </v-col>
            </v-row>
          </template>
        </v-col>
      </v-row>
    </v-container>
  </div>
</template>

<style lang="scss" scoped>
.border-highlight {
   border-bottom: 1px solid gray;
   border-top: 1px solid gray;
 }
</style>
