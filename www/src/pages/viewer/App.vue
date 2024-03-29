<template>
  <div class="absolute inset-0 bg-gray-500 overflow-clip" id="viewer">
    <TouchAnimation :default_radius="0.10*Math.min(width, height)"
                    @error="on_error" v-if="loaded">
      <ViewerCanvas v-if="model !== null" 
                    :model="model"
                    :current_element="element" :width="width" :height="height" @error="on_error" 
                    :camera_prop="camera" @update:CameraProp="(v) => {camera = v;}"
                    @hover:Element="({uuid}) => {on_hover_element(uuid);}"
                    @select:Element="({uuid}) => {on_select_element(uuid);}"
                    />
      <div class="fixed right-5 bottom-5 flex flex-row items-end justify-end">
        <TouchScrollTool :delta="touch_scroll_delta"
                       @error="on_error"
                       @click="(evt) =>   {on_touch_scroll_tool_click(evt);}"
                       @scroll="(evt) =>  {on_touch_scroll_tool_scroll(evt);}"
                       @press="(evt) =>   {on_touch_scroll_event('press', evt);}"
                       @release="(evt) => {on_touch_scroll_event('release', evt);}"
                       />
      </div>
    </TouchAnimation>
    <div class="fixed bottom-20 left-2">
      <OrbitTool  ref="default_tool" v-if="tool_name === 'default_tool' && camera !== null" 
                  v-model="camera"
                  @click="() => {tool_name = 'select_tool';}"
                  @error="on_error"
                  />
      <SelectTool ref="select_tool" v-if="tool_name === 'select_tool'" 
                  v-model="tool_name" :entries="menu_tool_entries" 
                  @error="on_error"
                  />
      <AddTool    ref="add_tool" v-if="tool_name === 'add_tool'" 
                  v-model="element" @add="on_add" @close="() => {tool_name = 'default_tool'; element = null;}" 
                  @error="on_error"
                  />
    </div>
    <div class="fixed bottom-5 left-2 flex flex-row gap-x-2 text-slate-900 hover:text-slate-800">
      <div class="cursor-pointer hover:text-slate-950" @click="on_new_model"><DocumentPlusIcon class="h-10 w-10" /></div>
      <div class="cursor-pointer hover:text-slate-950" @click="on_exit"><ArrowRightOnRectangleIcon class="h-10 w-10" /></div>
    </div>
    <div class="fixed top-2 left-2">
      <div class="flex flex-col">
        <div class="flex flex-row">
        <input class="text-slate-400 text-sm bg-transparent focus:outline-none" v-model="name" placeholder="Untitled" 
                 @focus="() => {name_editing = true;}"
                 @blur="() => {name_editing = false;}"
                 />
        <div v-if="name_editing === true"><a href="#"><CheckIcon class="text-slate-400 h-5 w-5" /></a></div>
        </div>
        <div class="text-slate-500 text-xs" v-if="save_state !== null">{{ save_state }}</div>
      </div>
    </div>
    <div class="fixed top-2 right-2">
      <div class="flex flex-col items-end">
        <div v-for="e in elements"
             :key="e.uuid"
             class="text-slate-500 text-xs font-light"
             :class="{
               'text-slate-400 font-normal': hover_element_uuid === e.uuid,
               'text-slate-200 font-normal': select_element_uuid === e.uuid,
             }">
          {{ e.type_name }} [{{ e.uuid }}]
        </div>
        <div class="mt-1 flex flex-col items-center text-slate-500 hover:text-slate-600" v-if="select_element_uuid === null">
          <div class="cursor-pointer rounded-md hover:text-slate-400 hover:bg-slate-600" 
               @click="on_add_element"><PlusIcon class="h-8 w-8" /></div>
        </div>
        <div class="mt-1 flex flex-col gap-y-2 items-center text-slate-500 hover:text-slate-600" v-if="select_element_uuid !== null">
          <div class="cursor-pointer rounded-md hover:text-slate-400 hover:bg-slate-600" 
               :class="{
                 'bg-slate-600 text-slate-400': tool_name === 'Pan',
               }"
               @click="select_tool_pan"
               ><PanIcon class="h-7 w-7" /></div>
          <div class="cursor-pointer rounded-md hover:text-slate-400 hover:bg-slate-600" 
               :class="{
                 'bg-slate-600 text-slate-400': tool_name === 'Rotate',
               }"
               @click="select_tool_rotate"
               ><ArrowPathIcon class="h-7 w-7" /></div>
          <div class="cursor-pointer rounded-md hover:text-slate-400 hover:bg-slate-600" 
               :class="{
                 'bg-slate-600 text-slate-400': tool_name === 'Scale',
               }"
               @click="select_tool_scale"
               ><ArrowTopRightOnSquareIcon class="h-7 w-7" /></div>
          <div class="cursor-pointer rounded-md hover:text-slate-400 hover:bg-slate-600" 
               @click="on_remove_element"
               ><TrashIcon class="h-7 w-7" /></div>
        </div>
      </div>
    </div>
  </div>
  <ModalErrorComposable :error="error" v-if="error !== null" @dismiss="error = null" />
</template>
<script>
import { computed } from 'vue';

import { useResizeable } from "@/extras/extra-vue-ui/composable/resizeable.js";
import { useError } from "@/extras/extra-vue-ui/composable/error.js";
import { useSaveable } from "@/extras/extra-vue-ui/composable/saveable.js";
import ModalErrorComposable from "@/extras/extra-vue-ui/modal/modalerrorcomposable.vue";
import ViewerCanvas from "./components/viewercanvas.vue";
import TouchAnimation from "@/extras/extra-vue-ui/touch/touchanimation.vue";
import TouchScrollTool from "@/extras/extra-vue-ui/touch/touchscrolltool.vue";
import SelectTool from "./components/selecttool.vue";
import AddTool from "./components/addtool.vue";
import OrbitTool from "./components/orbittool.vue";

import { Orbit } from "./tools/orbit.js";
import { Pan   } from "./tools/pan.js";
import { Rotate} from "./tools/rotate.js";
import { Scale } from "./tools/scale.js";

import { mount_wasm } from "@/js/mount.js";
import { save_model, read_model } from "@/js/storage.js";

import PanIcon from "@/icons/panicon.vue";
import { 
    ArrowPathIcon,
    ArrowRightOnRectangleIcon,
    ArrowTopRightOnSquareIcon,
    CheckIcon, 
    DocumentPlusIcon,
    MinusIcon,
    PlusIcon,
    TrashIcon,
    } from "@heroicons/vue/24/outline";

const MENU_TOOL_ENTRIES = [
  { value: 'default_tool',   label: 'Close' },
  { value: 'select_tool',    label: 'Select' },
  { value: 'add_tool',       label: 'Add' },
  { value: 'new_model',      label: 'New' },
  { value: 'exit',           label: 'Exit' },
  // { value: 'translate', label: 'Translate' },
  // { value: 'rotate',    label: 'Rotate' },
  // { value: 'stretch',   label: 'Stretch' },
];

function getGrid(wasm) {
  const grid = wasm.GridBuilder.new()
        .center([0.0, 0.0, 0.0])
        .normal([0.0, 0.0, 1.0])
        .tangent([1.0, 0.0, 0.0])
        .delta(0.1)
        .n(10)
        .build();
  return grid;
}

export default {
  name: "App",
  setup() {
    const { width, height, resize } = useResizeable("viewer");
    const { error, on_error, catcher } = useError();
    const { register_save_function, save, save_state } = useSaveable();
    return { width, height, resize, 
             error, on_error, catcher,
             register_save_function, save, save_state,
             };
  },
  provide() {
    return {
      wasm: computed(() => this.wasm),
    };
  },
  props: {
    mount_wasm: {
      type: Function,
      default: mount_wasm,
    },
  },
  components: {
    ModalErrorComposable,
    ViewerCanvas,
    TouchAnimation, TouchScrollTool,
    SelectTool, AddTool, OrbitTool,
    PanIcon,
    ArrowPathIcon,
    ArrowRightOnRectangleIcon,
    ArrowTopRightOnSquareIcon,
    CheckIcon, 
    DocumentPlusIcon,
    MinusIcon,
    PlusIcon,
    TrashIcon,
  },
  mounted() {
    this.catcher("mounted", 
    () => {
      this.resize();
      this.tool_ = new Orbit();
      this.mount_wasm()
      .then((wasm) => { this.wasm = wasm; })
      .then(() => { this.on_read(); })
      .then(() => { this.register_save_function(() => { this.on_save(); }); })
      .catch((e) => { this.on_error({msg: "Error in mount_wasm", e}); });
    });
  },
  data() { 
    return { 
      wasm: null, 
      tool_name_data: 'default_tool',
      menu_tool_entries: MENU_TOOL_ENTRIES,
      model: null,
      element: null,
      camera_data: null,
      name_editing: false,
      hover_element_uuid: null,
      select_element_uuid: null,
      tool_: null,
    }; 
  },
  computed: {
    elements: function() {
      return this.catcher("elements:get",
      () => {
        if (this.model === null) { return []; }
        let r = [];
        this.model.for_each_element((e) => { r.push({uuid: e.uuid(), type_name: e.type_name()}); });
        return r;
      });
    },
    camera: {
      get() { return this.camera_data; },
      set(v) {
        this.catcher("camera:set",
        () => {
          this.camera_data = v;
          if (this.model !== null && this.camera_data !== null) {
            const camera = this.camera_data.as_camera();
            const updated = camera.updated();
            this.model = this.model.set_camera(this.camera_data.as_camera());
            if (updated) { this.save(); }
          }
        });
      },
    },
    name: {
      get() { return this.model !== null ? this.model.name() : ""; },
      set(v) {
        this.catcher("name:set",
        () => {
          if (this.model !== null) {
            this.model = this.model.set_name(v);
            this.save();
          }
        });
      },
    },
    loaded: function() { return this.wasm !== null && this.model !== null; },
    tool: {
      get() { return this.tool_;
        // return this.$refs[this.tool_name]; 
      },
      set(v) { this.tool_ = v; }
    },
    touch_scroll_delta: function() { return 5.0; }, // console.log(this.tool); return this.tool === undefined || this.tool.scroll_delta === undefined ? 5.0 : this.tool.scroll_delta; },
    tool_name: function() {
      return (this.tool !== null) ? this.tool.name() : "";
    },
    /*
    tool_name: {
      get() { return this.tool_name_data; },
      set(v) { 
        if (v === 'exit') {
          window.location.href = '/';
        } else if (v === 'new_model') {
          this.on_new_model();
          this.tool_name_data = 'default_tool';
        } else {
          this.tool_name_data = v; 
        }
      },
    },
    */
  },
  methods: {
    select_tool_pan: function() { 
      this.catcher("select_tool_pan", () => { 
        if (this.tool.name() === "Pan") {
          this.select_tool_orbit();
        } else {
          this.tool = new Pan(this.wasm, this.select_element_uuid);
        }
      });
    },
    select_tool_rotate: function() { 
      this.catcher("select_tool_rotate", () => { 
        if (this.tool.name() === "Rotate") {
          this.select_tool_orbit();
        } else {
          this.tool = new Rotate(this.wasm, this.select_element_uuid);
        }
      });
    },
    select_tool_scale: function() { 
      this.catcher("select_tool_scale", () => { 
        if (this.tool.name() === "Scale") {
          this.select_tool_orbit();
        } else {
          this.tool = new Scale(this.wasm, this.select_element_uuid);
        }
      });
    },
    select_tool_orbit: function() {
      this.catcher("select_tool_orbit", () => { 
        this.tool = new Orbit(); 
      });
    },
    on_hover_element: function(uuid) {
      this.catcher("on_hover_element",
      () => {
          this.hover_element_uuid = uuid;
      });
    },
    on_select_element: function(uuid) {
      this.catcher("on_select_element",
      () => {
        if (uuid !== null) {
          if (this.select_element_uuid === uuid) {
            this.select_element_uuid = null;
            this.select_tool_orbit();
          } else {
            this.select_element_uuid = uuid;
            if (this.tool_name === "Pan") {
               this.tool = new Pan(this.wasm, this.select_element_uuid);
            } else if (this.tool_name === "Rotate") {
               this.tool = new Rotate(this.wasm, this.select_element_uuid);
            } else if (this.tool_name === "Scale") {
               this.tool = new Scale(this.wasm, this.select_element_uuid);
            } else if (this.tool_name !== "Orbit") {
              throw new Error(`Tool name ${this.tool_name} is not handled...`);
            }
          }
        }
      });
    },
    on_save: function() {
      this.catcher("on_save", 
      () => { 
        if (this.model !== null) { 
          save_model(this.model).catch((e) => { this.on_error({msg: "Error in save_model", e}); }); 
        } 
      });
    },
    on_read: function() {
      try {
        const sp = new URLSearchParams(window.location.search);
        if (sp.has("id")) {
          const model = read_model(this.wasm, sp.get("id"));
          if (model === null) { throw new Error(`Unable to retrieve model with id ${id}`); }
          this.model = model;
          this.camera = this.model.camera();
        } else {
          this.on_new_model();
        }
      } catch (e) {
        console.error(e);
        if (this.model === null) { this.on_new_model(); }
      }
    },
    on_new_model: function() {
      this.catcher("on_new_model",
      () => {
        if (this.wasm === null) { throw new Error("Wasm not initialised yet"); }
        let grid = getGrid(this.wasm);
        const model = this.wasm.ModelWasmed.default().add_element(grid);
        save_model(model)
        .then((model) => {
          const url = new URL(window.location); url.searchParams.set("id", model.id());
          window.location = url;
        })
        .catch((e) => {this.on_error("Error in on_new_model::save_model", e);});
      });
    },
    on_touch_scroll_tool_scroll: function(evt) {
      this.catcher("on_touch_scroll_tool_scroll",
      () => {
        if ('vibrate' in navigator) { navigator.vibrate(100); }
        // this.tool.scroll(evt);
        this.model = this.tool.scroll(evt, this.model);
        this.camera = this.model.camera();
      });
    },
    on_touch_scroll_tool_click: function(evt) {
      this.catcher("on_touch_scroll_tool_click", 
      () => {
        if ('vibrate' in navigator) { navigator.vibrate(250); }
        // this.tool.click(evt);
        this.model = this.tool.click(evt, this.model);
        this.camera = this.model.camera();
      });
    },
    on_touch_scroll_event: function(key, evt) {
      console.log("Event", key, evt);
    },
    on_add: function() {
      this.catcher("on_add",
      () => {
        this.model = this.model.add_element(this.element);
        this.save();
        this.element = null;
      });
    },
    on_remove_element: function() {
      this.catcher("on_remove_element",
      () => {
        const uuid = this.select_element_uuid;
        this.select_element_uuid = null;
        this.model = this.model.remove_element(uuid);
        this.save();
      });
    },
    on_add_element: function() {
      this.catcher("on_add_element",
      () => {
        const e = this.wasm.HexahedronBuilder.new()
          .start([0.0, 0.0, 0.0])
          .end([0.1, 0.1, 0.1])
          .build();
        this.select_element_uuid = e.uuid();
        this.model = this.model.add_element(e);
        this.save();
      });
    },
    on_exit: function() {
      this.catcher("on_exit",
      () => {
        window.location.href = '/';
      });
    },
  },
}
</script>

