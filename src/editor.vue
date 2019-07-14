<template>
  <div class="quill-editor">
    <input type="file" id="getFile" accept="image/x-png,image/gif,image/jpeg" @change="imageHandler($event)"  style="display: none;"/>
    <slot name="toolbar"></slot>
    <div ref="editor"></div>
  </div>
</template>

<script>
  // require sources
  import _Quill from 'quill'
  import defaultOptions from './options'
  import axios from 'axios'
  const Quill = window.Quill || _Quill

  // pollfill
  if (typeof Object.assign != 'function') {
    Object.defineProperty(Object, 'assign', {
      value(target, varArgs) {
        if (target == null) {
          throw new TypeError('Cannot convert undefined or null to object')
        }
        const to = Object(target)
        for (let index = 1; index < arguments.length; index++) {
          const nextSource = arguments[index]
          if (nextSource != null) {
            for (const nextKey in nextSource) {
              if (Object.prototype.hasOwnProperty.call(nextSource, nextKey)) {
                to[nextKey] = nextSource[nextKey]
              }
            }
          }
        }
        return to
      },
      writable: true,
      configurable: true
    })
  }

  // export
  export default {
    name: 'quill-editor',
    data() {
      return {
        _options: {},
        _content: '',
        defaultOptions
      }
    },
    props: {
      content: String,
      value: String,
      disabled: {
        type: Boolean,
        default: false
      },
      options: {
        type: Object,
        required: false,
        default: () => ({})
      },
      globalOptions: {
        type: Object,
        required: false,
        default: () => ({})
      }
    },
    mounted() {
      this.initialize()
    },
    beforeDestroy() {
      this.quill = null
      delete this.quill
    },
    methods: {
      // Init Quill instance
      initialize() {
        if (this.$el) {

          // Options
          this._options = Object.assign({}, this.defaultOptions, this.globalOptions, this.options)

          // Instance
          this.quill = new Quill(this.$refs.editor, this._options)
          
          this.quill.enable(false)

          // Set editor content
          if (this.value || this.content) {
            this.quill.pasteHTML(this.value || this.content)
          }

          // Disabled editor
          if (!this.disabled) {
            this.quill.enable(true)
          }

          // Mark model as touched if editor lost focus
          this.quill.on('selection-change', range => {
            if (!range) {
              this.$emit('blur', this.quill)
            } else {
              this.$emit('focus', this.quill)
            }
          })

          // Update model if text changes
          this.quill.on('text-change', (delta, oldDelta, source) => {
            let html = this.$refs.editor.children[0].innerHTML
            const quill = this.quill
            const text = this.quill.getText()
            const content = this.quill.getContents()
            if (html === '<p><br></p>') html = ''
            this._content = html
            this.$emit('textChange', { delta, oldDelta, source, content })
            this.$emit('input', this._content)
            this.$emit('change', { html, text, quill })
          })

          // Emit ready event
          this.$emit('ready', this.quill)
          this.options.modules.handlers = {
            'image': document.getElementById('getFile').click()
          }
        }
      },
      methods: {
        imageHandler(e) {
          const form = new FormData()
          form.append('image', e.target.files[0])
          // console.log(image); // Always true
          // console.log(callback); // Always undefined

          console.log(e.target.files[0])
          console.log(form)

          // Should get image in here somehow..
          // console.log(form)

          // Send image to server,
          //  Server will return link to image
          axios.defaults.withCredentials = true
          axios.post(this.options.uploadURL, form, this.options.options)
                  .then(function(cbData) {
                    console.log(cbData)
                    this.quill.container.insertEmbed(0, 'image', 'https://ed808.com/' + cbData.data.fileUrl);
                    // In here should add image tag to editor somehow..

                  })
                  .catch(function (error) {
                    console.log(error);
                  });
        },
      }
    },
    watch: {
      // Watch content change
      content(newVal, oldVal) {
        if (this.quill) {
          if (newVal && newVal !== this._content) {
            this._content = newVal
            this.quill.pasteHTML(newVal)
          } else if(!newVal) {
            this.quill.setText('')
          }
        }
      },
      // Watch content change
      value(newVal, oldVal) {
        if (this.quill) {
          if (newVal && newVal !== this._content) {
            this._content = newVal
            this.quill.pasteHTML(newVal)
          } else if(!newVal) {
            this.quill.setText('')
          }
        }
      },
      // Watch disabled change
      disabled(newVal, oldVal) {
        if (this.quill) {
          this.quill.enable(!newVal)
        }
      }
    }
  }
</script>
