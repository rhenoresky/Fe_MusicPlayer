<script setup>
defineProps({
  entitas: String,
  id: String,
})
const emit = defineEmits(['cancelDelete', 'successDelete'])

async function deleteItem(id, entitas) {
  await useFetch(`http://localhost:5000/${entitas}/${id}`, {
    method: 'DELETE'
  })

  emit('successDelete')
}
</script>

<template>
  <section class="fixed w-full h-full inset-0 bg-black/75 flex justify-center items-center z-50">
    <div class="h-[200px] w-[450px] bg-[#9DB2BF] rounded p-6 flex flex-col justify-center">
      <div class="flex gap-3">
        <div class="h-10 w-10 mt-2 rounded-full bg-red-200 flex justify-center items-center">
          <Icon name="cil:warning" size="24px" class="text-red-600 mb-1" />
        </div>
        <div class="w-[90%]">
          <slot></slot>
        </div>
      </div>
      <div class="flex justify-end gap-4 mt-2 mr-4">
        <button class="px-4 py-1 rounded text-slate-50 ring-1 ring-slate-50" @click="emit('cancelDelete')">Batal</button>
        <button class="px-4 py-1 rounded text-slate-50 ring-1 ring-slate-50 bg-red-500"
          @click="deleteItem(id, entitas)">Hapus</button>
      </div>
    </div>
  </section>
</template>