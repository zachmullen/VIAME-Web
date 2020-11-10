<script>
export default {
  name: 'AttributeSettings',
  inject: ['girderRest'],
  props: {
    show: {
      type: Boolean,
      default: false,
    },
    selectedAttribute: {
      type: Object,
      required: true,
    },
  },
  data: () => ({
    selectedIndex: undefined,
    name: '',
    belongs: '',
    datatype: '',
    values: [],
    addNew: false,
    changed: false,
  }),
  computed: {
    textValues: {
      get() {
        if (this.values) {
          return this.values.join('\n');
        }
        return '';
      },
      set(value) {
        this.values = value.split('\n');
      },
    },
  },
  watch: {
    name() {
      this.changed = true;
    },
    belongs() {
      this.changed = true;
    },
    datatype() {
      this.changed = true;
    },
    values() {
      this.changed = true;
    },
  },
  mounted() {
    this.name = this.selectedAttribute.name;
    this.belongs = this.selectedAttribute.belongs;
    this.datatype = this.selectedAttribute.datatype;
    this.values = this.selectedAttribute.values;
    if (!this.selectedAttribute._id.length) {
      this.addNew = true;
    } else {
      this.addNew = false;
    }
  },
  asyncComputed: {
    async attributes() {
      const { data } = await this.girderRest.get('/viame/attribute');
      return data;
    },
  },
  created() {
    this.setDefaultValue();
  },
  methods: {
    setDefaultValue() {
      this.name = '';
      this.belongs = 'track';
      this.datatype = 'number';
      this.values = [];
      this.$nextTick(() => {
        this.changed = false;
      });
    },
    add() {
      this.setDefaultValue();
      this.addNew = true;
      this.$refs.form.resetValidation();
    },
    async submit() {
      if (!this.$refs.form.validate()) {
        return;
      }

      const content = {
        name: this.name,
        belongs: this.belongs,
        datatype: this.datatype,
        values: this.datatype === 'text' ? this.values : [],
      };

      if (this.addNew) {
        await this.girderRest.post('/viame/attribute', content);
      } else {
        await this.girderRest.put(
          `/viame/attribute/${this.selectedAttribute._id}`,
          content,
        );
      }
      this.$emit('close');
      this.changed = false;
    },
    async deleteAttribute() {
      const result = await this.$prompt({
        title: 'Confirm',
        text: 'Do you want to delete this attribute?',
        confirm: true,
      });
      if (!result) {
        return;
      }
      await this.girderRest.delete(
        `/viame/attribute/${this.selectedAttribute._id}`,
      );
      this.$emit('close');
    },
  },
};
</script>

<template>
  <v-dialog
    v-model="show"
    max-width="350"
  >
    <v-card class="attribute-settings">
      <v-card-title class="pb-0">
        Attributes
        <v-card-text>
          <v-form ref="form">
            <v-text-field
              v-model="name"
              label="Name"
              :rules="[v => !!v || 'Name is required']"
              required
            />
            <v-select
              v-model="datatype"
              style="max-width: 220px;"
              :items="[
                { text: 'Boolean', value: 'boolean' },
                { text: 'Number', value: 'number' },
                { text: 'Text', value: 'text' }
              ]"
              label="Datatype"
            />
            <v-textarea
              v-if="datatype === 'text'"
              v-model="textValues"
              style="max-width: 250px;"
              label="Predefined values"
              hint="Line separated values"
              outlined
              auto-grow
              row-height="30"
            />
          </v-form>
          <v-card-actions>
            <v-row>
              <v-tooltip
                open-delay="100"
                bottom
              >
                <template #activator="{ on }">
                  <div v-on="on">
                    <v-btn
                      class="hover-show-child"
                      color="error"
                      :disabled="!selectedAttribute._id.length"
                      @click.prevent="deleteAttribute"
                    >
                      Delete
                    </v-btn>
                  </div>
                </template>
                <span
                  class="ma-0 pa-1"
                >
                  Deletion of Attribute
                </span>
              </v-tooltip>
              <v-spacer />
              <v-btn
                text
                @click="$emit('close')"
              >
                Cancel
              </v-btn>
              <v-btn
                color="primary"
                @click.prevent="submit"
              >
                Save
              </v-btn>
            </v-row>
          </v-card-actions>
        </v-card-text>
      </v-card-title>
    </v-card>
  </v-dialog>
</template>

<style lang="scss">
.attribute-settings {
  .v-textarea textarea {
    line-height: 24px;
  }
}
</style>
