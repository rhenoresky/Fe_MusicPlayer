<script setup>
const emit = defineEmits(['cancelEdit', 'successEdit'])
const props = defineProps({
  entitas: String,
  editValue: Object,
})

const classBox = ref({
  songs: 'w-[600px]',
  albums: 'w-[400px]',
})

async function EditItem(entitas, value) {
  let bodyMessage = {}
  if (entitas === 'songs') {
    bodyMessage = {
      title: value.title,
      year: value.year,
      performer: value.performer,
      genre: value.genre,
      duration: value.duration,
      albumId: value.albumId
    }
  } else {
    bodyMessage = {
      name: value.name,
      year: value.year
    }
  }
  await useFetch(`http://localhost:5000/${entitas}/${value.id}`, {
    method: 'PUT',
    body: bodyMessage
  })

  emit('successEdit')
}
</script>

<template>
  <section class="fixed w-full h-full inset-0 bg-black/75 flex justify-center items-center z-50">
    <form @submit.prevent="EditItem(entitas, editValue)" :class="classBox[entitas]"
      class="bg-[#9DB2BF] rounded p-8 flex flex-col justify-center items-center">
      <slot></slot>
      <div class="flex justify-center gap-4 mt-8">
        <button class="px-4 py-1 rounded text-slate-50 ring-1 ring-slate-50 bg-red-500"
          @click="emit('cancelEdit')">Batal</button>
        <button type="submit" class="px-4 py-1 rounded text-slate-50 ring-1 ring-slate-50 bg-green-500">Kirim</button>
      </div>
    </form>
  </section>
</template>