<script lang="ts">
import {
  computed, defineComponent, onBeforeMount, reactive, Ref, ref,
} from '@vue/composition-api';

import {
  useSelectedTrackId, useFrame, useTrackMap, useEditingMode, useTypeStyling, useAllTypes,
} from 'vue-media-annotator/provides';
import Track, { TrackId, Feature } from 'vue-media-annotator/track';
import TrackItem from 'vue-media-annotator/components/TrackItem.vue';

import { useApi, Attribute } from 'viame-web-common/apispec';
import AttributeInput from 'viame-web-common/components/AttributeInput.vue';
import AttributeEditor from 'viame-web-common/components/AttributeEditor.vue';

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
    AttributeEditor,
  },
  props: {
    lockTypes: {
      type: Boolean,
      default: false,
    },
  },
  setup(props) {
    const attributes = ref([] as Attribute[]);
    const editingAttribute: Ref<Attribute | null> = ref(null);
    const trackMap = useTrackMap();
    const editingModeRef = useEditingMode();
    const typeStylingRef = useTypeStyling();
    const allTypesRef = useAllTypes();

    const activeSettings = reactive({
      confidencePairs: false,
      trackAttributes: false,
      detectionAttributes: false,
    });

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
    const activeTrackAttributesCount = computed(() => trackAttributes.value.filter(
      (a) => selectedTrack.value && selectedTrack.value.attributes[a.name] !== undefined,
    ).length);
    const activeDetectionAttributesCount = computed(() => detectionAttributes.value.filter(
      (a) => selectedDetection.value && selectedDetection.value.attributes
        && selectedDetection.value.attributes[a.name] !== undefined,
    ).length);
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

    async function closeEditor() {
      editingAttribute.value = null;
      attributes.value = await getAttributes();
    }

    function trackAttributeAdd() {
      editingAttribute.value = {
        belongs: 'track',
        datatype: 'text',
        name: 'NewAttribute',
        _id: '',
      };
    }
    function editAttribute(attribute: Attribute) {
      editingAttribute.value = attribute;
    }
    function detectionAttributeAdd() {
      console.log('Add Track Attribute');
    }
    function confidencePairsAdd() {
      console.log('Add Confidence Pairs');
    }

    function toggleActiveSettings(
      type: 'confidencePairs' | 'trackAttributes' | 'detectionAttributes',
    ) {
      activeSettings[type] = !activeSettings[type];
    }

    onBeforeMount(async () => {
      attributes.value = await getAttributes();
    });

    return {
      selectedTrackIdRef,
      /* Attributes */
      detectionAttributes,
      trackAttributes,
      activeSettings,
      activeTrackAttributesCount,
      activeDetectionAttributesCount,
      editingAttribute,
      /* Selected */
      selectedDetection,
      selectedTrack,
      /* Update functions */
      closeEditor,
      editAttribute,
      toggleActiveSettings,
      updateFeatureAttribute,
      updateTrackAttribute,
      trackAttributeAdd,
      detectionAttributeAdd,
      confidencePairsAdd,
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
            <v-spacer />
            <v-subheader class="border-highlight">
              <v-row>
                <b>Confidence Pairs:</b>
                <v-spacer />
                <v-tooltip
                  v-if="activeSettings.confidencePairs"
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
                      @click="confidencePairsAdd"
                    >
                      <v-icon small>
                        mdi-plus
                      </v-icon>
                      Attribute
                    </v-btn>
                  </template>
                  <span>Add a new Confidence Pair</span>
                </v-tooltip>
                <v-tooltip
                  open-delay="200"
                  bottom
                  max-width="200"
                >
                  <template #activator="{ on }">
                    <v-btn
                      small
                      icon
                      class="pb-2 ml-2"
                      :color="activeSettings.confidencePairs ? 'accent': ''"
                      v-on="on"
                      @click="toggleActiveSettings('confidencePairs')"
                    >
                      <v-icon
                        small
                      >
                        mdi-pencil
                      </v-icon>
                    </v-btn>
                  </template>
                  <span>Edit ConfidencePairs</span>
                </v-tooltip>
              </v-row>
            </v-subheader>            <v-row
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
                <b>Track Attributes:</b>
                <v-spacer />
                <v-tooltip
                  v-if="activeSettings.trackAttributes"
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
                <v-tooltip
                  open-delay="200"
                  bottom
                  max-width="200"
                >
                  <template #activator="{ on }">
                    <v-btn
                      small
                      icon
                      class="pb-2 ml-2"
                      :color="activeSettings.trackAttributes ? 'accent': ''"
                      v-on="on"
                      @click="toggleActiveSettings('trackAttributes')"
                    >
                      <v-icon
                        small
                      >
                        mdi-pencil
                      </v-icon>
                    </v-btn>
                  </template>
                  <span>Edit Attributes</span>
                </v-tooltip>
              </v-row>
            </v-subheader>
            <v-row
              class="ma-0"
              style="max-height:30vh; overflow-y:auto; overflow-x:hidden"
              dense
            >
              <v-col
                v-if="activeSettings.trackAttributes ||activeTrackAttributesCount"
                class="pa-2"
              >
                <span
                  v-for="(attribute, i) of trackAttributes"
                  :key="i"
                >
                  <v-row
                    v-if="
                      activeSettings.trackAttributes || selectedTrack.attributes[attribute.name]"
                    class="ma-0"
                    dense
                    align="center"
                  >
                    <v-col
                      style="font-size:.8em"
                    >
                      {{ attribute.name }}:
                    </v-col>
                    <v-col class="px-1">
                      <AttributeInput
                        v-if="activeSettings.trackAttributes"
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
                      <div
                        v-else
                      >
                        {{ selectedTrack.attributes[attribute.name] }}
                      </div>
                    </v-col>
                    <v-col
                      v-if="activeSettings.trackAttributes"
                      cols="1"
                    >
                      <v-btn
                        icon
                        small
                        @click="editAttribute(attribute)"
                      >
                        <v-icon
                          small
                        >
                          mdi-settings
                        </v-icon>
                      </v-btn>
                    </v-col>
                  </v-row>
                </span>
              </v-col>
              <v-col v-else>
                <div>
                  No Track Attributes Set
                </div>
              </v-col>
            </v-row>
            <v-subheader
              v-if="selectedDetection"
              class="border-highlight"
            >
              <v-container
                dense
                class="px-0 my-2"
              >
                <v-row class="pt-2">
                  <b>Detection Attributes:</b>
                  <v-spacer />
                  <v-tooltip
                    v-if="activeSettings.detectionAttributes"
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
                  <v-tooltip
                    open-delay="200"
                    bottom
                    max-width="200"
                  >
                    <template #activator="{ on }">
                      <v-btn
                        small
                        icon
                        class="pb-2 ml-2"
                        :color="activeSettings.detectionAttributes ? 'accent': ''"
                        v-on="on"
                        @click="toggleActiveSettings('detectionAttributes')"
                      >
                        <v-icon
                          small
                        >
                          mdi-pencil
                        </v-icon>
                      </v-btn>
                    </template>
                    <span>Edit Attributes</span>
                  </v-tooltip>
                </v-row>
                <v-row class="mb-1">
                  {{ `Frame: ${selectedDetection.frame}` }}
                </v-row>
              </v-container>
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
              style="max-height:30vh; overflow-y:auto; overflow-x:hidden"
              dense
            >
              <v-col
                v-if="activeSettings.detectionAttributes || activeDetectionAttributesCount"
                class="pa-2"
              >
                <span
                  v-for="(attribute, i) of detectionAttributes"
                  :key="i"
                >
                  <v-row
                    v-if="activeSettings.detectionAttributes
                      || selectedDetection.attributes[attribute.name]"
                    class="ma-0"
                    dense
                    align="center"
                  >
                    <v-col style="font-size:.8em">
                      {{ attribute.name }}:
                    </v-col>
                    <v-col class="px-1">
                      <AttributeInput
                        v-if="activeSettings.detectionAttributes"
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
                      <div
                        v-else
                      >
                        {{ selectedDetection.attributes[attribute.name] }}
                      </div>
                    </v-col>
                    <v-col
                      v-if="activeSettings.detectionAttributes"
                      cols="1"
                    >
                      <v-btn
                        icon
                        small
                        @click="editAttribute(attribute)"
                      >
                        <v-icon
                          small
                        >
                          mdi-settings
                        </v-icon>
                      </v-btn>
                    </v-col>
                  </v-row>
                </span>
              </v-col>
              <v-col v-else>
                <div>
                  No Detection Attributes Set
                </div>
              </v-col>
            </v-row>
          </template>
        </v-col>
      </v-row>
    </v-container>
    <attribute-editor
      v-if="editingAttribute != null"
      :show="editingAttribute != null"
      :selected-attribute="editingAttribute"
      @close="closeEditor"
    />
  </div>
</template>

<style lang="scss" scoped>
.border-highlight {
   border-bottom: 1px solid gray;
   border-top: 1px solid gray;
 }
</style>
