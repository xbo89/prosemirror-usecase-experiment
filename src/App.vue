<template>
  <div id="editor" style="margin-bottom: 23px"></div>

  <div style="display: none" id="content">
    <h3>Hello ProseMirror</h3>

    <p>This is editable text. You can focus it and start typing.</p>

    <p>
      To apply styling, you can select a piece of text and manipulate its
      styling from the menu. The basic schema supports <em>emphasis</em>,
      <strong>strong text</strong>,
      <a href="http://marijnhaverbeke.nl/blog">links</a>,
      <code>code font</code>, and <img src="/img/smiley.png" /> images.
    </p>

    <p>
      Block-level structure can be manipulated with key bindings (try
      ctrl-shift-2 to create a level 2 heading, or enter in an empty textblock
      to exit the parent block), or through the menu.
    </p>

    <p>
      Try using the “list” item in the menu to wrap this paragraph in a numbered
      list.
    </p>
  </div>
</template>

<script setup>
import { onMounted } from "vue";
// 导入必要的库和组件
import { EditorState, NodeSelection, TextSelection } from "prosemirror-state";
import { EditorView } from "prosemirror-view";
import { Schema, DOMParser } from "prosemirror-model";
import { addListNodes } from "prosemirror-schema-list";
import { exampleSetup } from "prosemirror-example-setup";
import { schema } from "prosemirror-schema-basic";
import { dropCursor } from "prosemirror-dropcursor";

// 扩展基本schema以支持拖拽按钮
const mySchema = new Schema({
  nodes: addListNodes(schema.spec.nodes, "paragraph block*", "block"),
  marks: schema.spec.marks,
});

onMounted(() => {
  window.view = new EditorView(document.querySelector("#editor"), {
    state: EditorState.create({
      doc: DOMParser.fromSchema(mySchema).parse(
        document.querySelector("#content")
      ),
      plugins: [
        ...exampleSetup({ schema: mySchema }),
        dropCursor({ width: 2, color: "red" }),
      ],
    }),
    nodeViews: {
      paragraph: (node, view, getPos) =>
        new SelectableParagraphView(node, view, getPos),
    },
  });
  const editorEl = document.querySelector("#editor");
  editorEl.addEventListener("drop", (event) => {
    event.preventDefault();
    const data = event.dataTransfer.getData("text/plain");

    // Check if the data is not an empty string
    if (data) {
      try {
        const parsedData = JSON.parse(data);

        if (parsedData.type === "paragraph") {
          const pos = window.view.posAtCoords({
            left: event.clientX,
            top: event.clientY,
          });
          const transaction = window.view.state.tr.insert(
            pos.pos,
            schema.nodes.paragraph.create(null, schema.text(parsedData.content))
          );
          window.view.dispatch(transaction);

          // Store the position of the newly inserted node
          window.view.newNodePos = pos.pos;
        }
      } catch (error) {
        console.error("Invalid JSON data:", error);
      }
    }
  });
});

const createDragHandle = () => {
  const dragHandle = document.createElement("span");
  dragHandle.className = "drag-handle";
  dragHandle.textContent = "☰";
  dragHandle.draggable = true;

  dragHandle.addEventListener("dragstart", (event) => {
    event.dataTransfer.setData("text/plain", ""); // for Firefox
    event.dataTransfer.effectAllowed = "move";
    event.dataTransfer.setDragImage(dragHandle, 0, 0);
    // closest(event.target, ".paragraph").classList.add("dragging");
  });

  dragHandle.addEventListener("dragend", (event) => {
    // closest(event.target, ".paragraph").classList.remove("dragging");
  });
  return dragHandle;
};

class SelectableParagraphView {
  constructor(node, view, getPos) {
    this.node = node;
    this.view = view;
    this.getPos = getPos;
    const dragHandle = createDragHandle();
    this.dom = document.createElement("div");
    this.contentDOM = document.createElement("p");
    this.dom.draggable = true;
    this.dom.appendChild(dragHandle);
    this.dom.appendChild(this.contentDOM);
    this.prevSelection = null;

    // this.dom.addEventListener("dragstart", (event) => {
    //   this.prevSelection = this.view.state.selection;
    //   event.dataTransfer.setData(
    //     "text/plain",
    //     JSON.stringify({ type: "paragraph", content: node.textContent })
    //   );
    //   event.dataTransfer.effectAllowed = "move";
    // });

    // this.dom.addEventListener("dragend", () => {
    //   const { state } = this.view;
    //   const tr = state.tr.setSelection(this.prevSelection);
    //   this.view.dispatch(tr);
    // });
    this.dom.addEventListener("dragend", () => {
      const { state } = this.view;
      // Check if newNodePos is defined
      if (typeof this.view.newNodePos !== "undefined") {
        const tr = state.tr.setSelection(
          TextSelection.create(state.doc, this.view.newNodePos)
        );
        this.view.dispatch(tr);
      }
    });
    // this.dom.addEventListener("dragend", () => {
    //   const { state } = this.view;
    //   const tr = state.tr.setSelection(
    //     TextSelection.create(state.doc, this.view.newNodePos)
    //   );
    //   this.view.dispatch(tr);
    // });
    dragHandle.addEventListener("mousedown", () => {
      const { state } = this.view;
      this.prevSelection = this.view.state.selection;
      const tr = state.tr.setSelection(
        new NodeSelection(state.doc.resolve(this.getPos()))
      );
      this.view.dispatch(tr);
    });
    // dragHandle.addEventListener("dragend", () => {
    //   const { state } = this.view;
    //   const tr = state.tr.setSelection(this.prevSelection);
    //   this.view.dispatch(tr);
    // });
  }

  stopEvent(event) {
    return event.type === "mousedown";
  }

  ignoreMutation() {
    return true;
  }
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}
.drag-handle,
.select-button {
  display: inline;
  cursor: pointer;
}
.paragraph {
  position: relative;
}
.drag-handle-container,
.select-button-container {
  position: absolute;
  left: -24px;
  top: 0;
}
.select-button-container {
  left: -48px;
}
.ProseMirror-selectednode {
  background-color: lightblue;
  outline: none;
  border-radius: 6px;
}
</style>
