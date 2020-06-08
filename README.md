# Epub.js Tips

Tips and tricks for using [Epub.js](https://github.com/futurepress/epub.js/)

Works with Epub.js v0.3.

Note: all the snippets here assuems `ePub`, `book = ePub()`, and `rendition = book.renderTo()` in scope. We also make use of a debounce function, an implementation of which is:

```js
const debounce = (f, wait, immediate) => {
    let timeout
    return (...args) => {
        const later = () => {
            timeout = null
            if (!immediate) f(...args)
        }
        const callNow = immediate && !timeout
        clearTimeout(timeout)
        timeout = setTimeout(later, wait)
        if (callNow) f(...args)
    }
}
```

## Navigation

### Scroll through pages when using paginated layout

Handling horizontal scrolling means that you can use horizontal touchpad swipes to turn the pages.

```js
const rtl = book.package.metadata.direction === 'rtl'
const goLeft = rtl ? () => rendition.next() : () => rendition.prev()
const goRight = rtl ? () => rendition.prev() : () => rendition.next()

const onwheel = debounce(event => {
    const { deltaX, deltaY } = event
    if (Math.abs(deltaX) > Math.abs(deltaY)) {
        if (deltaX > 0) goRight()
        else if (deltaX < 0) goLeft()
    } else {
        if (deltaY > 0) rendition.next()
        else if (deltaY < 0) rendition.prev()
    }
    event.preventDefault()
}, 100, true)
document.documentElement.onwheel = onwheel
```

## Selection

### Basic selection operations

```js
const getSelections = () => rendition.getContents()
    .map(contents => contents.window.getSelection())

const clearSelection = () => getSelections().forEach(s => s.removeAllRanges())

const selectByCfi = cfi => getSelections().forEach(s => s.addRange(rendition.getRange(cfi)))
```

### Get position of selected text

```js
const getRect = (target, frame) => {
    const rect = target.getBoundingClientRect()
    const viewElementRect =
        frame ? frame.getBoundingClientRect() : { left: 0, top: 0 }
    const left = rect.left + viewElementRect.left
    const right = rect.right + viewElementRect.left
    const top = rect.top + viewElementRect.top
    const bottom = rect.bottom + viewElementRect.top
    return { left, right, top, bottom }
}

rendition.hooks.content.register((contents, /*view*/) => {
    const frame = contents.document.defaultView.frameElement
    contents.document.onclick = e => {
        const selection = contents.window.getSelection()
        const range = selection.getRangeAt(0)
        const { left, right, top, bottom } = getRect(range, frame)
        // Note: besides a range object, the same method can also be used to get the position of any element
    }
})
```

### Select across pages when using paginated layout

Go to the next page when selecting to the end of a page, this makes it possible for the user to select across multiple pages:

```js
rendition.on('selected', debounce(cfiRange => {
    const selCfi = new ePub.CFI(cfiRange)
    selCfi.collapse()
    const compare = CFI.compare(selCfi, rendition.location.end.cfi) >= 0
    if (compare) rendition.next()
}, 1000))
```

## Working with CFIs

### Get CFI from a link

```js
const getCfiFromHref = async href => {
    const id = href.split('#')[1]
    const item = book.spine.get(href)
    await item.load(book.load.bind(book))
    const el = id ? item.document.getElementById(id) : item.document.body
    return item.cfiFromElement(el)
}
```

### Get range CFI from two CFIs

```js
// adapted from https://github.com/futurepress/epub.js/blob/be24ab8b39913ae06a80809523be41509a57894a/src/epubcfi.js#L502
const makeRangeCfi = (a, b) => {
    const start = CFI.parse(a), end = CFI.parse(b)
    const cfi = {
        range: true,
        base: start.base,
        path: {
            steps: [],
            terminal: null
        },
        start: start.path,
        end: end.path
    }
    const len = cfi.start.steps.length
    for (let i = 0; i < len; i++) {
        if (CFI.equalStep(cfi.start.steps[i], cfi.end.steps[i])) {
            if (i == len - 1) {
                // Last step is equal, check terminals
                if (cfi.start.terminal === cfi.end.terminal) {
                    // CFI's are equal
                    cfi.path.steps.push(cfi.start.steps[i])
                    // Not a range
                    cfi.range = false
                }
            } else cfi.path.steps.push(cfi.start.steps[i])
        } else break
    }
    cfi.start.steps = cfi.start.steps.slice(cfi.path.steps.length)
    cfi.end.steps = cfi.end.steps.slice(cfi.path.steps.length)

    return 'epubcfi(' + CFI.segmentString(cfi.base)
        + '!' + CFI.segmentString(cfi.path)
        + ',' + CFI.segmentString(cfi.start)
        + ',' + CFI.segmentString(cfi.end)
        + ')'
}
```

## Other

### Re-render annotations

Annotations are often rendered wrong when, for example, the rendition is resized, or after applying a theme. The solution is to re-render them whenever any of that happens.

```js
const redrawAnnotations = () =>
    rendition.views().forEach(view => view.pane ? view.pane.render() : null)

rendition.on('rendered', redrawAnnotations)

/* when applying themes */
function applyTheme(themeName, stylesheet) {
    rendition.themes.register(themeName, stylesheet)
    rendition.themes.select(themeName)
    redrawAnnotations()
}
```

### Fix current location lost after resizing

To fix location drift when resizing multiple times in a row, we keep a `location` that doesn't change when rendition has just been resized. Then, when the resize is done, we correct the location with it. But this correction will itself trigger a `relocated` event, so we create a further `correcting` variable to track this.

```js
let location
let justResized = false
let correcting = false
rendition.on('relocated', () => {
    // console.log('relocated')
    if (!justResized) {
        if (!correcting) {
            // console.log('real relocation')
            location = rendition.currentLocation().start.cfi
        } else {
            // console.log('corrected')
            correcting = false
        }
    } else {
        // console.log('correcting')
        justResized = false
        correcting = true
        rendition.display(location)
    }
})
rendition.on('resized', () => {
    // console.log('resized')
    justResized = true
})
```
