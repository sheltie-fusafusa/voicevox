<template>
  <q-dialog
    v-model="ToolBarCustomDialogOpenComputed"
    maximized
    transition-show="jump-up"
    transition-hide="jump-down"
    class="tool-bar-custom-dialog transparent-backdrop"
  >
    <q-layout container view="hHh Lpr fFf" class="bg-background">
      <q-page-container class="root">
        <q-header class="q-py-sm">
          <q-toolbar>
            <q-toolbar-title class="text-display"
              >ツールバーのカスタマイズ</q-toolbar-title
            >
            <q-space />
            <q-btn
              unelevated
              color="toolbar-button"
              text-color="toolbar-button-display"
              class="text-no-wrap text-bold q-mr-sm"
              :disable="isDefault"
              @click="applyDefaultSetting"
              >デフォルトに戻す</q-btn
            >
            <q-btn
              unelevated
              color="toolbar-button"
              text-color="toolbar-button-display"
              class="text-no-wrap text-bold q-mr-sm"
              :disable="!isChanged"
              @click="saveCustomToolbar"
              >保存</q-btn
            >
            <!-- close button -->
            <q-btn
              round
              flat
              icon="close"
              color="display"
              @click="finishOrNotDialog"
            />
          </q-toolbar>
        </q-header>
        <q-page>
          <q-card flat square class="preview-card">
            <q-toolbar class="bg-toolbar preview-toolbar">
              <draggable
                v-model="toolbarButtons"
                :item-key="toolbarButtonKey"
                @start="toolbarButtonDragging = true"
                @end="toolbarButtonDragging = false"
              >
                <template
                  #item="{ element: button }: { element: ToolbarButtonTagType }"
                >
                  <q-btn
                    unelevated
                    color="toolbar-button"
                    text-color="toolbar-button-display"
                    :class="
                      (button === 'EMPTY' ? ' radio-space' : ' radio') +
                      ' text-no-wrap text-bold q-mr-sm'
                    "
                  >
                    {{ getToolbarButtonName(button) }}
                    <q-tooltip
                      :delay="800"
                      anchor="center right"
                      self="center left"
                      transition-show="jump-right"
                      transition-hide="jump-left"
                      :style="{
                        display: toolbarButtonDragging ? 'none' : 'block',
                      }"
                      >{{ usableButtonsDesc[button] }}</q-tooltip
                    >
                  </q-btn>
                </template>
              </draggable>
              <div class="preview-toolbar-drag-hint">
                ドラッグでボタンの並びを変更できます。
              </div>
            </q-toolbar>

            <q-card-actions>
              <div class="text-h5">表示するボタンの選択</div>
            </q-card-actions>
            <q-card-actions class="no-padding">
              <q-list class="usable-button-list bg-surface">
                <q-item
                  v-for="(desc, key) in usableButtonsDesc"
                  :key="key"
                  v-ripple
                  tag="label"
                >
                  <q-item-section>
                    <q-item-label>{{ getToolbarButtonName(key) }}</q-item-label>
                    <q-item-label caption>{{ desc }}</q-item-label>
                  </q-item-section>
                  <q-item-section avatar>
                    <q-toggle v-model="toolbarButtons" :val="key" />
                  </q-item-section>
                </q-item>
              </q-list>
            </q-card-actions>
          </q-card>
        </q-page>
      </q-page-container>
    </q-layout>
  </q-dialog>
</template>

<script setup lang="ts">
import { computed, ref, watch, Ref } from "vue";
import draggable from "vuedraggable";
import { useStore } from "@/store";
import { ToolbarButtonTagType, ToolbarSettingType } from "@/type/preload";
import { getToolbarButtonName } from "@/store/utility";

const props =
  defineProps<{
    modelValue: boolean;
  }>();
const emit =
  defineEmits<{
    (e: "update:modelValue", val: boolean): void;
  }>();

const store = useStore();

// computedだと値の編集ができないが、refにすると起動時に読み込まれる設定が反映されないので、watchしている
const toolbarButtons = ref([...store.state.toolbarSetting]);
const toolbarButtonKey = (button: ToolbarButtonTagType) => button;
const toolbarButtonDragging = ref(false);
const selectedButton: Ref<ToolbarButtonTagType | undefined> = ref(
  toolbarButtons.value[0]
);
watch(
  () => store.state.toolbarSetting,
  (newData) => {
    // このwatchはToolbar Setting更新時にも機能するが、
    // 以下の処理はVOICEVOX起動時のみ機能してほしいので、toolbarButtonsのlengthが0の時だけ機能させる
    if (!toolbarButtons.value.length) {
      toolbarButtons.value = [...newData];
      selectedButton.value = newData[0];
    }
  }
);

const defaultSetting: ToolbarSettingType = [];
window.electron.getDefaultToolbarSetting().then((setting) => {
  defaultSetting.push(...setting);
});

const usableButtonsDesc: Record<ToolbarButtonTagType, string> = {
  PLAY_CONTINUOUSLY:
    "選択されているテキスト以降のすべてのテキストを読み上げます。",
  STOP: "テキストが読み上げられているときに、それを止めます。",
  EXPORT_AUDIO_SELECTED:
    "選択されているテキストの読み上げを音声ファイルに書き出します。",
  EXPORT_AUDIO_ALL:
    "入力されているすべてのテキストの読み上げを音声ファイルに書き出します。",
  EXPORT_AUDIO_CONNECT_ALL:
    "入力されているすべてのテキストの読み上げを一つの音声ファイルに繋げて書き出します。",
  SAVE_PROJECT: "プロジェクトを上書き保存します。",
  UNDO: "操作を一つ戻します。",
  REDO: "元に戻した操作をやり直します。",
  IMPORT_TEXT: "テキストファイル(.txt)を読み込みます。",
  EMPTY:
    "これはボタンではありません。レイアウトの調整に使います。また、実際には表示されません。",
};

const ToolBarCustomDialogOpenComputed = computed({
  get: () => props.modelValue || isChanged.value,
  set: (val) => emit("update:modelValue", val),
});

const isChanged = computed(() => {
  const nowSetting = store.state.toolbarSetting;
  return (
    toolbarButtons.value.length != nowSetting.length ||
    toolbarButtons.value.some((e, i) => e != nowSetting[i])
  );
});
const isDefault = computed(() => {
  return (
    toolbarButtons.value.length == defaultSetting.length &&
    toolbarButtons.value.every((e, i) => e == defaultSetting[i])
  );
});

// ボタンが追加されたときはそれをフォーカスし、
// 削除されたときは一番最初のボタンをフォーカスするようにする
watch(
  () => toolbarButtons.value,
  (newData, oldData) => {
    if (oldData.length < newData.length) {
      selectedButton.value = newData[newData.length - 1];
    } else if (
      selectedButton.value != undefined &&
      oldData.includes(selectedButton.value) &&
      !newData.includes(selectedButton.value)
    ) {
      selectedButton.value = newData[0];
    }
  }
);

const applyDefaultSetting = async () => {
  const result = await store.dispatch("SHOW_CONFIRM_DIALOG", {
    title: "ツールバーをデフォルトに戻します",
    message: "ツールバーをデフォルトに戻します。<br/>よろしいですか？",
    html: true,
    actionName: "はい",
    cancel: "いいえ",
  });
  if (result === "OK") {
    toolbarButtons.value = [...defaultSetting];
    selectedButton.value = toolbarButtons.value[0];
  }
};
const saveCustomToolbar = () => {
  store.dispatch("SET_TOOLBAR_SETTING", {
    data: [...toolbarButtons.value],
  });
};

const finishOrNotDialog = async () => {
  if (isChanged.value) {
    const result = await store.dispatch("SHOW_WARNING_DIALOG", {
      title: "カスタマイズを終了しますか？",
      message: "このまま終了すると、カスタマイズは破棄されてリセットされます。",
      actionName: "終了",
    });
    if (result === "OK") {
      toolbarButtons.value = [...store.state.toolbarSetting];
      selectedButton.value = toolbarButtons.value[0];
      ToolBarCustomDialogOpenComputed.value = false;
    }
  } else {
    selectedButton.value = toolbarButtons.value[0];
    ToolBarCustomDialogOpenComputed.value = false;
  }
};
</script>

<style lang="scss" scoped>
@use '@/styles/variables' as vars;
@use '@/styles/colors' as colors;

.tool-bar-custom-dialog .q-layout-container :deep(.absolute-full) {
  right: 0 !important;
  overflow-x: hidden;

  > .scroll {
    width: unset !important;
    overflow: hidden;
  }
}

.preview-toolbar {
  height: calc(#{vars.$toolbar-height} + 8px);
  display: block;
}

// draggableのdiv。
.preview-toolbar > div:not(.preview-toolbar-drag-hint) {
  width: 100%;
  display: inline-flex;
}

.preview-toolbar-drag-hint {
  padding-top: 8px;
  padding-bottom: 8px;
}

.preview-card {
  width: 100%;
  min-width: 460px;
  background: var(--color-background);
}

.usable-button-list {
  // menubar-height + toolbar-height * 2(main+preview) + window-border-width
  // 52(preview part buttons) * 2 + 46(select part title) + 22(preview part hint)
  height: calc(
    100vh - #{vars.$menubar-height + (vars.$toolbar-height) +
      vars.$window-border-width + 52px + 46px + 22px}
  );
  width: 100%;
  overflow-y: scroll;
}

.radio {
  &:hover {
    cursor: grab;
  }
}

.radio-space {
  @extend .radio;
  flex-grow: 1;
  color: transparent;
}
</style>
