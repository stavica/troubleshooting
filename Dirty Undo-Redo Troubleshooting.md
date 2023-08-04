**Adding console log statements to the extension workflow history file**
- Didn't seem to see a direct correlation going on. I'd see a message that history is saved when I click a node and then click anything that isn't that same node. Just opening up the list didn't seem to really specifically trigger something. 
- Still, the issue only exists when the Dirty Undo/Redo extension has it's javascript files loaded. It also seems much more reasonable to try to edit the extension's code to not break a component of the main code, rather than modify the main code so some extension doesn't break it.
	- However, maybe any lessons learned here can help make comfyUI more resilient/adaptable? 
	- Adjusting the core file to work with the extension can shine a light on what the extension is doing that breaks the core file's function. 
		- Potentially something other extension-makers can keep in mind to ensure their code doesn't produce the same issues.

**A quick note on what didn't work**
- Any attempts to change where the selectedIndex or selectedItem variables are, or how they're called, doesn't seem to impact anything other than us no longer seeing the currently-picked model highlighted when we see the list.

**Sorta-fix**
- Changes to the initial part of `requestAnimationFrame`  function in `contextMenuFilter.js` in `/ComfyUI/web/extensions/core/` are what have made it workable with the extension active.
- Ideally, changes should be made to the actual extension's files.

**Code: Initial part of the `requestAnimationFrame` function**
> [!info] Original
> ```js
> 				requestAnimationFrame(() => {
> 					const currentNode = LGraphCanvas.active_canvas.current_node;
> 					const clickedWidget = currentNode.widgets
> 						.filter(w => w.type === "combo" && w.options.values.length === values.length)
> 						.find(w => w.options.values.every((v, i) => v === values[i]));
> 				
> 					if (clickedWidget !== undefined) {
> 						const clickedComboValue = clickedWidget.value;
> 						let selectedIndex = values.findIndex(v => v === clickedComboValue);
> 						let selectedItem = displayedItems?.[selectedIndex];
> 						updateSelected();
> ```

> [!note] Iteration 1: Allows searchbar to be used, highlighting function still broken
> ```js
> requestAnimationFrame(() => {
>     const currentNode = LGraphCanvas.active_canvas.current_node;
>     const clickedWidget = currentNode.widgets
>         .filter(w => w.type === "combo" && w.options.values.length === values.length)
>         .find(w => w.options.values.every((v, i) => v === values[i]));
> 
>     if (clickedWidget !== undefined) {
>         const clickedComboValue = clickedWidget.value;
>         let selectedIndex = values.findIndex(v => v === clickedComboValue);
>         let selectedItem = displayedItems?.[selectedIndex];
> 
>         // Apply highlighting to the selected item
>         function updateSelected() {
>             selectedItem?.style.setProperty("background-color", "");
>             selectedItem?.style.setProperty("color", "");
>             selectedItem = displayedItems[selectedIndex];
>             selectedItem?.style.setProperty("background-color", "#ccc", "important");
>             selectedItem?.style.setProperty("color", "#000", "important");
>         }
> 
>         updateSelected();
>     }
> ```

> [!note] Iteration 2: Fully functional, except for when it randomly breaks completely
> ```js
> 				requestAnimationFrame(() => {
> 					const currentNode = LGraphCanvas.active_canvas.current_node;
> 					const clickedWidget = currentNode.widgets
> 						.filter(w => w.type === "combo" && w.options.values.length === values.length)
> 						.find(w => w.options.values.every((v, i) => v === values[i]));
> 				
> 					// Apply highlighting to the selected item
> 					function updateSelected() {
> 						selectedItem?.style.setProperty("background-color", "");
> 						selectedItem?.style.setProperty("color", "");
> 						selectedItem = displayedItems[selectedIndex];
> 						selectedItem?.style.setProperty("background-color", "#ccc", "important");
> 						selectedItem?.style.setProperty("color", "#000", "important");
> 					}
> 				
> 					if (clickedWidget !== undefined) {
> 						const clickedComboValue = clickedWidget.value;
> 						selectedIndex = values.findIndex(v => v === clickedComboValue);
> 						selectedItem = displayedItems?.[selectedIndex];
> 				
> 						updateSelected();
> 					}
> ```

- I haven't actually tested if iteration 1 does or doesn't randomly break completely. It could be that it does too.

**When iteration 2 'breaks completely'**
I refer to the whole functionality of the `Filter list` searchbar textinput as the 'Filter'.

<mark style="background: #FFB86CA6;">Orange</mark>: Issue, non-showstopper
<mark style="background: #FF5582A6;">Red</mark>: Showstopper
Green: Something that 'reverses' the showstopper. I.e. it works again.

Tested so far:
- Load Default, click ckpt_name. Scroll, type, arrow key, enter:
	- No errors called up. <mark style="background: #FFB86CA6;">However, no item starts off highlighted.</mark> 
		- I can however use the left, or right arrow key to highlight the first or last model, and start from there. But still, <mark style="background: #FFB86CA6;">the selected model isn't starting off as highlighted</mark>.
	- When I type, the input causes the first item on the list to be highlighted.
	- Once an item is highlighted, I can use the up and down arrow keys to scroll through selected items. Left arrow key highlights the top item, right arrow key highlights the bottom item.
- Type a name until the desired model is selected, hit enter.
	- Works mostly as intended. <mark style="background: #FFB86CA6;">When clicking `cpkt_name` after doing this, that model is not highlighted. Same as if I refreshed/used `load default`.</mark>
- Click input, half-type a word, arrow key to select a random model then refresh page: No issue
- Created a new node by click+dragging from the 'clip' input for `Load Checkpoint`, typed some text into it.  Move the `Load Checkpoint` node around a bit. No issue with the menu.
- Alt tab to type the above bullet point. Return to the page and click `ckpt_name`, and <mark style="background: #FF5582A6;">the filter is broken. </mark> `Uncaught TypeError: can't access property "filter", currentNode.widgets is undefined`
	- Go back to type the next, above bullet point in Obsidian, and then check again. <mark style="background: #BBFABBA6;">The filter is now working again.</mark>
- Hit refresh, click `ckpt_name`, <mark style="background: #FF5582A6;">filter is broken</mark>.  `Uncaught TypeError: can't access property "filter", currentNode.widgets is undefined`
- Move the screen a bit, click `ckpt_name` again. <mark style="background: #BBFABBA6;">Filter is working.</mark>
	- We're getting somewhere, I think. Whenever I click a node, or make a change, then click somewhere on the canvas, that is when the `after()` function is triggered in 
- Zooming out to a certain level and then trying to open the menu <mark style="background: #FF5582A6;">breaks the filter.</mark> <mark style="background: #BBFABBA6;">Except when it doesn't.</mark> 
	- Zooming DOES NOT break the filter, but when a filter is broken, then whatever zoom you were in when it happened will cause the filter to be broken while you are in that zoom. 
		- Ex. Zoom a few paces in, some external/unknown factor breaks the filter when you click the filter. Zoom out or farther in one pace and no problem. Return to that first zoom level, and the issue arises again.


> [!info] The `before()` and `after()` functions in `WorkflowHistory.js`, from the undo/redo extension
> I've added two console log lines just to make it visible in the console log when the extension is 'saving' the interface's state. It seems to do this after majority of interactions, but not technically all.
> ```js
>     before() {
>         if (!this.enabled) return;
>         //console.log("before");
> 
>         const timestamp = Date.now();
>         if (timestamp - this.prev_undo_timestamp < this.state_merge_threshold) {
>             this.prev_undo_timestamp = timestamp;
>             //console.log("merged") // will be kept the same
>             return; //state is whithin the merge thesh and is discarded, don't do anything else
>         }
> 
>         // potential state outside merge thresh. note that it is not guaranteed to be pushed        
>         this.get_new_candidate_state(app);
>         console.log("Before save state");
>     }
> 
>     after() {
>         if (!this.enabled) return;
>         //console.log("after");
> 
>         {
>             // was there any change? if not, don't try to store the state
>             // (this check is not guaranteed to work due to node order, but I am reordering nodes onSelected;
>             //  so unless there are other operations that reorder nodes, it should be fine... I think)
>             const equal = this.equal_states(JSON.stringify(app.graph.serialize(), null), this.temp_state);
>             if (equal) return;
>         }
> 
>         this.tryAddToUndoHistory();
>         console.log("After save state");
>     }
> ```

**Adding more console log statements**
The closest thing to an answer I've gotten is that when the context menu is being opened at the same time that the undo/redo extension is saving a state, it breaks. Keep in mind, it only gets this far because of modifications made to the original code to begin with.

These current node options aren't fully expanded, because they can expand basically forever.

```ad-info
title: Current Node: No filter issues.

Current Node: When working.
```js
Current node:  
Object { inputs: (1) […], size: Float32Array(2), widgets: (1) […], serialize_widgets: true, getExtraMenuOptions: getExtraMenuOptions(_, options), type: "SaveImage", title: "Save Image", properties: {}, properties_info: [], flags: {}, … }
​
bgcolor: undefined
​
flags: Object {  }
​​
<prototype>: Object { … }
​
getExtraMenuOptions: function getExtraMenuOptions(_, options)
​
graph: Object { status: 1, last_node_id: 10, last_link_id: 10, … }
​
id: 9
​
inputs: Array [ {…} ]
​
mode: 0
​
onMouseDown: function onMouseDown(event)
​
onResize: function onResize()
​
order: 6
​
pos: Array [ 1451, 189 ]
​
properties: Object {  }
​
properties_info: Array []
​
serialize_widgets: true
​
size: Float32Array [ 210, 58 ]
​
title: "Save Image"
​
type: "SaveImage"
​
widgets: Array [ {…} ]
​
widgets_values: Array [ "ComfyUI" ]
​
<prototype>: Object { comfyClass: "SaveImage", getExtraMenuOptions: getExtraMenuOptions(_, options), setSizeForImage: setSizeForImage(), … }
contextMenuFilter.js:31:14

```


```ad-info
title: Current Node: When filter is broken


Current Node: When not working. 
```js
Current node:  
Object { inputs: (2) […], size: Float32Array(2), outputs: (1) […], serialize_widgets: true, getExtraMenuOptions: getExtraMenuOptions(_, options), type: "VAEDecode", title: "VAE Decode", properties: {…}, properties_info: (1) […], flags: {}, … }
​
bgcolor: undefined
​
flags: Object {  }
​
getExtraMenuOptions: function getExtraMenuOptions(_, options)
​
graph: Object { status: 1, last_node_id: 10, last_link_id: 10, … }
​
id: 8
​
inputs: Array [ {…}, {…} ]
​
mode: 0
​
onMouseDown: function onMouseDown(event)
​
onResize: function onResize()
​
order: 5
​
outputs: Array [ {…} ]
​
pos: Array [ 1209, 188 ]
​
properties: Object { "Node name for S&R": "VAEDecode" }
​
properties_info: Array [ {…} ]
​
serialize_widgets: true
​
size: Float32Array [ 210, 46 ]
​
title: "VAE Decode"
​
type: "VAEDecode"
​
<prototype>: Object { comfyClass: "VAEDecode", getExtraMenuOptions: getExtraMenuOptions(_, options), setSizeForImage: setSizeForImage(), … }
contextMenuFilter.js:31:14


```


At different zooms, 'current node:' is giving me a different existing node's information. Right now, it's breaking when it's on the Vae Decode node. Zoom in further, it's showing the KSampler node. Zoom out, it's on the `Save Image` node.