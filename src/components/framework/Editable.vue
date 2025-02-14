<template>
  <div
    :contenteditable="editable !== false"
    :disabled="editable === false"
    @keydown="onKeyDown"
    @input="changed = true"
    @focus="onFocus"
    @blur="onBlur"
    @click="onClick"
    @wheel="onWheel"
  ></div>
</template>

<script lang="ts">
import { Component, Vue, Watch } from 'vue-property-decorator'
import { countDecimals } from '@/services/productsService'

@Component({
  name: 'Editable',
  props: ['content', 'step', 'min', 'max', 'editable', 'disabled']
})
export default class extends Vue {
  private content: string
  private min: number
  private max: number
  private disabled: boolean
  private clickAt: number
  private changed = false
  private position: number
  private _incrementSelectionTimeout: number
  private _emitTimeout: number

  mounted() {
    const el = this.$el as HTMLElement

    el.innerText = this.content
  }

  @Watch('content')
  onContentChange() {
    if ((this.$el as HTMLElement).innerText !== this.content) {
      ;(this.$el as HTMLElement).innerText = this.content
    }
  }

  getCaretPosition() {
    let caretPos = 0
    let sel
    let range

    if (window.getSelection) {
      sel = window.getSelection()
      if (sel.rangeCount) {
        range = sel.getRangeAt(0)
        if (range.commonAncestorContainer.parentNode == this.$el) {
          caretPos = range.endOffset
        }
      }
    }

    return caretPos
  }

  selectAll() {
    window.setTimeout(() => {
      let sel: Selection
      let range: Range

      if (window.getSelection && document.createRange) {
        range = document.createRange()
        range.selectNodeContents(this.$el)
        sel = window.getSelection()
        sel.removeAllRanges()
        sel.addRange(range)
      } else if ((document.body as any).createTextRange) {
        range = (document.body as any).createTextRange()
        ;(range as any).moveToElementText(this.$el)
        ;(range as any).select()
      }
    }, 1)
  }

  onBlur(event) {
    if (event.which === 13 && !isNaN(event.target.innerText)) {
      event.preventDefault()
      return
    }

    if (this.changed) {
      event.target.innerHTML = event.target.innerText
      this.$emit('output', event.target.innerText)
    }

    if (window.getSelection) {
      window.getSelection().removeAllRanges()
    } else if ((document as any).selection) {
      ;(document as any).selection.empty()
    }
  }

  onKeyDown(event) {
    if (this.disabled || event.which === 13) {
      event.preventDefault()
      ;(this.$el as HTMLInputElement).blur()

      event.target.innerText = this.content

      return
    }

    if (event.which === 38 || event.which === 40) {
      this.increment((event.which === 40 ? 1 : -1) * (event.shiftKey ? 10 : 1))
    }
  }

  onFocus() {
    this.changed = false
  }

  onClick() {
    const now = Date.now()

    if (this.clickAt && now - this.clickAt < 150) {
      this.selectAll()
    }

    this.clickAt = now
  }

  increment(direction: number) {
    const parts = (this.$el as HTMLElement).innerText.trim().split(',')

    let text
    let partIndex
    let count = 0
    let position

    if (parts.length > 0) {
      if (this._incrementSelectionTimeout) {
        clearTimeout(this._incrementSelectionTimeout)
      }

      if (this.position) {
        position = this.position
      } else {
        position = this.getCaretPosition()
      }

      for (let i = 0; i < parts.length; i++) {
        const part = parts[i]
        if (position >= count && position <= count + part.length) {
          text = part.replace(/[^0-9-.]/g, '')
          partIndex = i
          break
        }

        count += part.length + 1
      }
    } else {
      text = parts[0]
    }

    if (isNaN(text as any)) {
      return
    }

    const max = typeof this.max === 'undefined' ? Infinity : this.max
    const min = typeof this.min === 'undefined' ? -Infinity : this.min
    const precision = countDecimals(text)
    const step = 1 / Math.pow(10, precision)
    const change = step * direction * -1

    text = Math.max(min, Math.min(max, +text + change)).toFixed(precision)

    if (parts.length > 1) {
      this.position = position

      parts[partIndex] = text
      text = parts.join(',')

      this._incrementSelectionTimeout = setTimeout(() => {
        this._incrementSelectionTimeout = null

        let sel

        if ((document as any).selection) {
          sel = (document as any).selection.createRange()
          sel.moveStart('character', position)
          sel.select()
        } else {
          sel = window.getSelection()
          sel.collapse(this.$el.lastChild, position)
        }

        this.position = null
      }, 100)
    }

    ;(this.$el as any).innerText = text

    if (this._emitTimeout) {
      clearTimeout(this._emitTimeout)
    }
    this._emitTimeout = setTimeout(() => {
      this._emitTimeout = null
      this.$emit('output', text)
    }, 50)
  }

  onWheel(event) {
    const focusedElement = document.activeElement as HTMLElement

    if (focusedElement !== event.target || !focusedElement.isContentEditable) {
      return
    }

    event.preventDefault()

    this.increment(Math.sign(event.deltaY) * (event.shiftKey ? 10 : 1))
  }
}
</script>
