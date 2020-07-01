<template>
  <div class="textimg__split">
    <div class="textimg__text">
      <h2>
        <k-editable
          ref="title"
          :breaks="false"
          :code="false"
          :placeholder="'Add a title…'"
          :content="text.title"
          @input="storeTitle"
        />
      </h2>
      <k-editable
        ref="text"
        :placeholder="$t('editor.blocks.image.caption.placeholder') + '…'"
        :breaks="true"
        :code="false"
        :content="content"
        @input="storeText"
        @enter="onEnter"
        @split="onEnter"
      />
    </div>
    <figure>
      <template v-if="attrs.src">
        <div
          ref="element"
          :style="style"
          :data-responsive="attrs.ratio"
          class="k-editor-textimg-block-wrapper"
          tabindex="0"
          @keydown.delete="replace"
          @keydown.enter="$emit('append')"
          @keydown.up="$emit('prev')"
          @keydown.down="$emit('next')"
        >
          <img ref="image" :src="attrs.src" :key="attrs.src" @dblclick="selectFile" @load="onLoad">
        </div>
        <figcaption>
          <k-editable
            :content="attrs.caption"
            :breaks="true"
            :placeholder="$t('editor.blocks.image.caption.placeholder') + '…'"
            :spellcheck="spellcheck"
            @prev="focus"
            @shiftTab="focus"
            @tab="$emit('next', $event)"
            @next="$emit('next', $event)"
            @split="$emit('append')"
            @enter="$emit('append')"
            @input="caption"
          />
        </figcaption>
      </template>
      <template v-else>
        <k-dropzone
          ref="element"
          class="k-editor-textimg-block-placeholder"
          tabindex="0"
          @keydown.native.delete="$emit('remove')"
          @keydown.native.enter="$emit('append')"
          @keydown.native.up.prevent="$emit('prev')"
          @keydown.native.down.prevent="$emit('next')"
          @drop="onDrop"
        >
          <k-button icon="upload" @click="uploadFile" @keydown.enter.native.stop>{{ $t('editor.blocks.image.upload') }}</k-button>
          {{ $t('editor.blocks.image.or') }}
          <k-button icon="image" @click="selectFile" @keydown.enter.native.stop>{{ $t('editor.blocks.image.select') }}</k-button>
        </k-dropzone>
      </template>
    </figure>

    <k-files-dialog ref="fileDialog" @submit="insertFile($event)" />
    <k-upload ref="fileUpload" @success="insertUpload" />

    <k-dialog ref="settings" @submit="saveSettings" size="medium">
      <k-form :fields="fields" v-model="attrs" @submit="saveSettings" />
    </k-dialog>

  </div>
</template>

<script>
export default {
  icon: "image",
  props: {
    content: String,
    title: String,
    text: {
      type: Object,
      default() {
        return {};
      }
    },
    attrs: {
      type: Object,
      default() {
        return {};
      }
    },
    endpoints: Object,
    spellcheck: Boolean
  },
  computed: {
    style() {
      if (this.attrs.ratio) {
        return 'padding-bottom:' + 100 / this.attrs.ratio + '%';
      }
    },
    fields() {
      return {
        alt: {
          label: this.$t('editor.blocks.image.alt.label'),
          type: "text",
          icon: "text"
        },
        link: {
          label: this.$t('editor.blocks.image.link.label'),
          type: "text",
          icon: "url",
          placeholder: this.$t('editor.blocks.image.link.placeholder')
        },
        css: {
          label: this.$t('editor.blocks.image.css.label'),
          type: "text",
          icon: "code",
        }
      };
    }
  },
  methods: {
    onEnter(html) {
      let newthis = this.$refs.text;
      newthis.dispatch(
        newthis.tr()
          .replaceSelectionWith(newthis.schema().nodes.hard_break.create())
          .scrollIntoView()
      );

      
    },
    storeTitle(html) {
      console.log(html);
      this.$emit("input", {
        title: html
      });
    },
    storeText(html) {
      this.$emit("input", {
        content: html
      });
    },
    caption(html) {
      this.input({
        caption: html
      });
    },
    edit() {
      if (this.attrs.guid) {
        this.$router.push(this.attrs.guid);
      }
    },
    focus() {
      if (this.attrs.src) {
        this.$refs.element.focus();
      } else {
        this.$refs.element.$el.focus();
      }
    },
    input(data) {
      this.$emit("input", {
        attrs: {
          ...this.attrs,
          ...data
        }
      });
    },
    storeText(data) {
      this.$emit("input", {
        text: {
          ...this.text,
          ...data
        }
      });
    },
    fetchFile(link) {
      this.$api.get(link).then(response => {
        this.input({
          guid: response.link,
          src: response.url,
          id: response.id,
          ratio: response.dimensions.ratio
        });
      });
    },
    insertFile(files) {
      const file = files[0];
      this.fetchFile(file.link);
    },
    insertUpload(files, response) {
      this.fetchFile(response[0].link);
      this.$events.$emit("file.create");
      this.$events.$emit("model.update");
      this.$store.dispatch("notification/success", ":)");
    },
    menu() {

      if (this.attrs.src) {
        return [
          {
            icon: "open",
            label: this.$t("editor.blocks.image.open.browser"),
            click: this.open
          },
          {
            icon: "edit",
            label: this.$t("editor.blocks.image.open.panel"),
            click: this.edit,
            disabled: !this.attrs.guid
          },
          {
            icon: "cog",
            label: this.$t("editor.blocks.image.settings"),
            click: this.$refs.settings.open
          },
          {
            icon: "image",
            label: this.$t("editor.blocks.image.replace"),
            click: this.replace
          },
        ];
      } else {
        return [];
      }

    },
    open() {
      window.open(this.attrs.src);
    },
    onDrop(files) {
      this.$refs.fileUpload.drop(files, {
        url: window.panel.api + "/" + this.endpoints.field + "/upload",
        multiple: false,
        accept: "image/*"
      });
    },
    onLoad() {
      const image = this.$refs.image;

      if (!this.attrs.ratio && image && image.width && image.height) {
        this.input({
          ratio: image.width / image.height
        });
      }
    },
    replace() {
      this.$emit("input", {
        attrs: {}
      });
    },
    selectFile() {
      this.$refs.fileDialog.open({
        endpoint: this.endpoints.field + "/files",
        multiple: false,
        selected: [this.attrs.id]
      });
    },
    settings() {
      this.$refs.settings.open();
    },
    saveSettings() {
      this.$refs.settings.close();
      this.input(this.attrs);
    },
    uploadFile() {
      this.$refs.fileUpload.open({
        url: window.panel.api + "/" + this.endpoints.field + "/upload",
        multiple: false,
        accept: "image/*"
      });
    },
  }
};
</script>

<style lang="scss">
.textimg__split {
  display: flex;
  justify-content: space-between;
   > * {
     flex: 0 0 45%;
   }
   figure {
    img {
      max-width: 100%;
    }
   }
  .k-editable,
  .k-editable .ProseMirror {
    height: 100%;
  }
  .k-upload {
    display: none;
  }
}
.k-editor-textimg-block-wrapper {
  padding-bottom: 0!important;
}

</style>
