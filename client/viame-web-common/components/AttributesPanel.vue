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
      editingModeRef,
      trackType,
      typeStylingRef,
      allTypesRef,
    };
  },
});
</script>

<template>
<v-container>
  <v-row
    class="attributes-panel"
  >
    <v-col
      style="overflow-y: auto;"
    >
      <v-subheader>Track Editor</v-subheader>
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
        <v-subheader>Confidence pairs:</v-subheader>
        <v-row
          dense
          style="max-height:20%"
        >
          <v-col>
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
        <v-subheader>
          <v-container>
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
                    :color="newTrackColor"
                    v-on="on"
                    @click="trackAdd"
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
          </v-container>
        </v-subheader>
        <v-row
          style="max-height:20%; overflow-y:auto"
        >
          <v-col>
            <v-row
              v-for="(attribute, i) of trackAttributes"
              :key="i"
              class="ml-1"
              dense
              align="center"
            >
              <v-col style="font-size:.8em">
                {{ attribute.name }}:
              </v-col>
              <v-col class="px-1">
                <AttributeInput
                  :datatype="attribute.datatype"
                  :name="attribute.values ? attribute.datatype : `${attribute.datatype}`"
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
                  class="mr-2"
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
      <v-col style="overflow-y: auto;">
        <v-subheader>Detection attributes</v-subheader>
        <div
          v-if="!selectedDetection"
          class="ml-4 body-2"
        >
          No detection selected
        </div>
        <template v-else>
          <div
            class="mx-4 body-2"
            style="line-height:22px;"
          >
            <div>Frame: {{ selectedDetection.frame }}</div>
            <div v-if="selectedDetection.fishLength">
              Fish length: {{ selectedDetection.fishLength }}
            </div>
            <AttributeInput
              v-for="(attribute, i) of detectionAttributes"
              :key="i"
              :datatype="attribute.datatype"
              :name="attribute.name"
              :values="attribute.values ? attribute.values : null"
              :value="
                (selectedDetection && selectedDetection.attributes)
                  ? selectedDetection.attributes[attribute.name]
                  : undefined
              "
              @change="updateFeatureAttribute(selectedTrackIdRef, selectedDetection, $event)"
            />
          </div>
        </template>
      </v-col>
    </v-col>
  </v-row>
</v-container>
</template>
