---
category: framework-deep-dive
---

# Clipboard

The clipboard feature (implemented by the {@link module:clipboard/clipboard~Clipboard} plugin) is responsible for integrating with the native clipboard – a feature of the operating system and the browser used when the user copies/cuts/pastes or drag&drop a content within the editor or from/to the "outside".

CKEditor 5 intercepts all the native events like `copy`, `cut` or `drop` and handles them on its side. The goal is to not allow the browser to touch the content in the editor which would lead to the browser messing it up.

There are two directions in which the content is processed:

* When a content is being pasted or dropped into the editor it goes through the [input pipeline](#input-pipeline).
* When a content is being copied, cut or dragged from the editor it goes through the [output pipeline](#output-pipeline).

Both pipelines allow the features to process the content to be inserted or set to the clipboard as well as overriding the default mechanisms at different stages of those processes.

## Input pipeline

When the user pastes or drops a content into the editor the browser fires an event which is intercepted by the clipboard feature and which originates this mechanism:

1. {@link module:clipboard/clipboardobserver~ClipboardObserver} turns that event into a synthetic {@link module:engine/view/document~Document#event:paste `view.Document#paste`} or {@link module:engine/view/document~Document#event:drop `view.Document#drop`}.
2. Since the content to be inserted by both actions (paste and drop) should usually be processed in the same way and both actions have very simillar effect, both events are turned into a single {@link module:engine/view/document~Document#event:clipboardInput `view.Document#clipboardInput`} event for easier handling.
3. Next, the clipboard feature listens to the `view.Document#clipboardInput` event, retrieves and pre-process the `text/html` or `text/plain` content which it finds in the {@link module:clipboard/datatransfer~DataTransfer event's `dataTransfer`} and fires the {@link module:clipboard/clipboard~Clipboard#event:inputTransformation `Clipboard#inputTransformation`} event with the retrieved content.
4. Finally, the clipboard feature listens to the `Clipboard#inputTransformation` event, takes the processed content and {@link module:engine/model/model~Model#insertContent inserts} it into the editor.

The clipboard feature listens to the `view.Document#clipboardInput` and `Clipboard#inputTransformation` events using low priority listeners. This means that adding a normal listener and calling `evt.stop()` allows overriding the behavior implemented by the clipboard feature. It is a similar mechanism to DOM's `evt.preventDefault()` that lets you override the default browser's behavior.

### Handling clipboard input differently

By default the clipboard feature retrieves `text/html` or `text/plain` from the clipboard, normalizes that data a bit (e.g. cleans up a [mess with whitespaces](https://github.com/ckeditor/ckeditor5-clipboard/issues/2)), converts that to a {@link module:engine/view/documentfragment~DocumentFragment view `DocumentFragment`} and fires the `Clipboard#inputTransformation` event with that document fragment for further processing.

The {@link module:engine/view/document~Document#event:clipboardInput `view.Document#clipboardInput`} event can be used to override this behavior. For example, you can use it to:

* Handle pasted/droppped files (which you can retrieve from the `dataTransfer`).

	Handling file upload requires, however, a lot more than reading {@link module:clipboard/datatransfer~DataTransfer#files `dataTransfer.files`} so for a complete code we recommend checking the source code of plugins like [`ImageUploadEditing`](https://github.com/ckeditor/ckeditor5-image/blob/master/src/imageupload/imageuploadediting.js).
* Change the type of data which the clipboard features reads from the clipboard. For instance, you may want to use `application/rtf` if it is present in the `dataTransfer` (and ignore `text/html` in that case).

	```js
	editor.editing.view.document.on( 'clipboardInput', ( evt, data ) => {
		const dataTransfer = data.dataTransfer;
		const rtfContent = dataTransfer.getData( 'application/rtf' );

		// If no RTF pasted, abort and let the clipboard feature handle the input.
		if ( !rtfContent ) {
			return;
		}

		// Convert an RTF raw string to a view document fragment.
		const viewContent = convertRtfStringToView( rtfContent );

		// Just like the clipboard feature, trigger the inputTransformation event
		// to allow further processing of the content.
		this.fire( 'inputTransformation', { content: viewContent, dataTransfer } );

		editor.editing.view.scrollToTheSelection();
		evt.stop();
	} );
	```

### Processing input content

The {@link module:clipboard/clipboard~Clipboard#event:inputTransformation `view.Document#inputTransformation`} event lets you process the content which is going to be inserted into the editor.

The default action is to {@link module:engine/model/model~Model#insertContent insert} the content (`data.content`, represented by a {@link module:engine/view/documentfragment~DocumentFragment}) to an editor if the data is not empty.

At this stage the pasted content can be processed by the features. E.g. a feature which wants to transform a pasted text into a link can be implemented in this way:

```js
editor.plugins.get( 'Clipboard' ).on( 'inputTransformation', ( evt, data ) => {
	if ( data.content.childCount == 1 && isUrlText( data.content.getChild( 0 ) ) ) {
		const linkUrl = data.content.getChild( 0 ).data;

		data.content = new ViewDocumentFragment( [
			ViewElement(
				'a',
				{ href: linkUrl },
				[ new ViewText( linkUrl ) ]
			)
		] );
	}
} );
```

The default action (inserting the content into the editor) is performed by a low priority listener, so it can be overridden by a normal one. With the `lowest` priority you can also execute actions after the content was already inserted.

```js
editor.plugins.get( 'Clipboard' ).on( 'inputTransformation', ( evt, data ) => {
	console.log( 'Content was inserted.' );
}, { priority: 'lowest' } );
```

## Output pipeline

The output pipeline is the equivalent of the input pipeline but for the copy and cut operations.
It allows to process the content which will be then put into the clipboard or to override the whole process.

### 1. On {@link module:engine/view/document~Document#event:copy `view.Document#copy`} and {@link module:engine/view/document~Document#event:cut `view.Document#cut`}

The default action is to:

1. {@link module:engine/model/model~Model#getSelectedContent get selected content} from the editor,
2. prevent the default action of the native `copy` or `cut` event,
3. fire {@link module:engine/view/document~Document#event:clipboardOutput `view.Document#clipboardOutput`} with a clone of the selected content converted to a {@link module:engine/view/documentfragment~DocumentFragment view document fragment}.

### 2. On {@link module:engine/view/document~Document#event:clipboardOutput `view.Document#clipboardOutput`}

The default action is to put the content (`data.content`, represented by a {@link module:engine/view/documentfragment~DocumentFragment}) to the clipboard as HTML. In case of the cut operation, the selected content is also deleted from the editor.

This action is performed by a low priority listener, so it can be overridden by a normal one.

At this stage the copied/cut content can be processed by other features.