<script setup lang="ts">
import { ref, onMounted } from "vue";
import { showToast } from "../utils";
import { useToast } from "primevue/usetoast";
import { compilePage } from "../main";

const toast = useToast();

const onSave = () => {
  let matched = 0;
  const datas: [() => boolean, () => void][] = [];

  for (const [condition, callback] of datas) {
    if (condition()) {
      callback();
      matched++;
    }
  }

  if (matched === 0) {
    showToast.warning(toast, "No changes", "No changes were made");
  }
};

onMounted(() => {
  compilePage();
});
</script>

<template>
  <div class="container">
    <Toast />
    <Card>
      <template #title> Settings </template>
      <template #content>
        <div class="card-inside">
          <span class="color-[gray] text-left">Author: xiaodong2008</span>
        </div>
      </template>
    </Card>
  </div>
</template>
