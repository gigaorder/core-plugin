<template>
  <v-flex :class="flex" class="px-2">
    <v-combobox v-model="value" :items="options" :label="field.label || field.key" clearable
                @change="onChange"
                :menu-props="{'z-index': 1000, 'closeOnContentClick': true}">
      <v-icon slot="append-outer" color="blue darken-1" @click.stop="toggleEditor">edit</v-icon>
    </v-combobox>
    <v-dialog v-model="showEditorDialog" max-width="80%">
      <v-card style="height: 80vh">
        <v-card-text style="height: calc(100% - 55px)">
          <monaco-editor :value.sync="code" :language="currentLanguage"
                         :label="model ? model.component : ''"
          ></monaco-editor>
        </v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn color="blue darken-1" flat @click.stop="toggleEditor">Close</v-btn>
          <v-btn color="blue darken-1" flat @click.stop="save">Save</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
    <v-snackbar
        v-model="showSnackbar"
        :timeout="3000"
        top right
    >
      {{ snackbarMsg }}
    </v-snackbar>
  </v-flex>
</template>

<script>
  import cms from 'cms';
  import MonacoEditor from 'MonacoEditor';

  const vueLanguage = 'vue';
  export default {
    props: ['model', 'field', 'noFlex'],
    created() {
    },
    data() {
      return {
        items: null,
        loading: false,
        code: '',
        currentLanguage: vueLanguage,
        showEditorDialog: false,
        showSnackbar: false,
        snackbarMsg: '',
      };
    },
    inject: {
      rootModel: { default: null },
      path: { default: null },
      noLayout: { default: null }
    },
    computed: {
      flex() {
        return this.noLayout ? 'xs-12' : this.field.flex;
      },
      options() {
        const list = cms.pluginComponents;
        return list.map(item => ({ text: item, value: item }));
      },
      value: {
        get() {
          return { text: this.model[this.field.key] || '' };
        },
        set(v) {
          this.model[this.field.key] = v.text;
        }
      }
    },
    mounted() {
      this.value = { text: this.model[this.field.key] };
      if (this.model) {
        this.loadPlugin()
          .then((items) => {
            const plugin = this.getPlugin(items);
            Object.assign(this.model, {
              url: plugin.url,
              path: plugin.path,
              pluginName: plugin.pluginName
            });
          }).finally(() => {
          this.getFileContent().then();
        });
      }
    },
    methods: {
      onChange(e) {
        if (this.field.onChange) this.field.onChange(e.value, this.rootModel, this.model);
      },
      getPlugin(items) {
        for (let i = 0; i < items.length; i++) {
          if (items[i].name !== 'dist') {
            if (items[i].name === this.model.component + '.vue') {
              return items[i];
            }
            if (items[i].children) {
              const result = (this.getPlugin(items[i].children));
              if (result) {
                return result;
              }
            }
          }
        }
      },
      loadPlugin() {
        return new Promise((resolve, reject) => {
          cms.socket.emit('loadPlugin', (tree) => {
            this.items = tree;
            resolve(tree);
          });
        });
      },
      getFileContent() {
        return new Promise(res => {
          this.loading = true;
          console.log(cms.baseUrl);
          fetch(cms.baseUrl + this.model.url)
            .then(res => res.text())
            .then(json => {
              this.$nextTick(() => {
                this.code = json;
                res(json);
              });
              this.loading = false;
            })
            .catch(() => {
              this.loading = false;
            });
        });
      },
      toggleEditor() {
        this.showEditorDialog = !this.showEditorDialog;
      },
      save() {
        cms.socket.emit('savePlugin', this.model.pluginName, this.model.path, this.code, (err) => {
          if (err) {
            this.snackbarMsg = err.message;
            this.showSnackbar = true;
          }
          this.snackbarMsg = 'Saved successfully';
          this.showSnackbar = true;
        });
        this.loadPlugin().then(() => {
          this.showEditorDialog = false;
        });
      }
    }
  };
</script>

<style>
</style>