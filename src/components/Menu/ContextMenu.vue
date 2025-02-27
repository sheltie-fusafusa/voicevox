<template>
  <q-menu
    ref="contextMenu"
    touch-position
    context-menu
    :no-focus="noFocus"
    transition-show="none"
    transition-hide="none"
  >
    <q-list dense>
      <q-item v-if="header" dense class="bg-background">
        <q-item-section class="text-weight-bold">{{ header }}</q-item-section>
      </q-item>
      <menu-item
        v-for="(menu, index) of menudata"
        :key="index + 1"
        :menudata="menu"
        :disable="
          (menu.type !== 'separator' && menu.disabled) ||
          (uiLocked && menu.type !== 'separator' && menu.disableWhenUiLocked)
        "
      >
      </menu-item>
    </q-list>
  </q-menu>
</template>

<script setup lang="ts">
import { computed, onMounted, onUnmounted, ref } from "vue";
import { QMenu } from "quasar";
import MenuItem from "./MenuItem.vue";
import { MenuItemButton, MenuItemSeparator } from "./type";
import { useStore } from "@/store";

defineProps<{
  header?: string;
  menudata: ContextMenuItemData[];
}>();
defineExpose({
  hide: () => {
    contextMenu.value?.hide();
  },
});
const store = useStore();
const uiLocked = computed(() => store.getters.UI_LOCKED);

const contextMenu = ref<QMenu>();
/**
 * コンテキストメニューがフォーカスを奪うかどうかを制御する。
 * 通常はアクセシビリティ考慮のためにフォーカスが移るが、input要素の場合は文字の選択範囲が非表示になってしまう。
 * この挙動が不要だと考えられるユーザーに向けてはフォーカスを奪わないようにする。
 */
const noFocus = ref(false);

const buttonCapturer = (event: Event) => {
  if (event instanceof PointerEvent) {
    // キーボードから開いた場合はアクセシビリティ(tabキーでの操作)を考慮してフォーカスされる
    // マウスなどから開いた場合は選択範囲の非表示回避のためにフォーカスされない
    noFocus.value = event.button !== -1;
  }
};
onMounted(() => {
  parent.addEventListener("contextmenu", buttonCapturer, { capture: true });
});
onUnmounted(() => {
  parent.removeEventListener("contextmenu", buttonCapturer);
});

export type ContextMenuItemData = MenuItemSeparator | MenuItemButton;
</script>
