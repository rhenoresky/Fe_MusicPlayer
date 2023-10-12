<script setup>
import image1 from '/img/4.png'
import image2 from '/img/5.png'
import image3 from '/img/Colab.jpg'

const watchValue = ref(0)
const {data: allMusic} = await useFetch('http://localhost:5000/songs')
const {data: listAlbum} = await useFetch('http://localhost:5000/albums', {
  watch: [watchValue]
})
const startIndexAlbum = ref(0)
const menuAlbums = reactive({})
function openMenuAlbums(key) {
  for (let item in menuAlbums) {
    if (item === key) {
      menuAlbums[item] = !menuAlbums[item]
    } else {
      menuAlbums[item] = false
    }
  }
}
const newAlbums = computed(() => {
  const albums = listAlbum.value.data.albums.slice(startIndexAlbum.value, startIndexAlbum.value + 4)
  for (let item of albums) {
    const name = item.id
    menuAlbums[name] = false
  }
  return albums
})
const listMusic = computed({
  get() {
    const item = []
    function hitungDurasi(durasi) {
      const menit = (Math.floor(durasi / 60))
      const detik = durasi % 60

      if (detik < 10) {
        return `${menit}:0${detik}`
      }
      return `${menit}:${detik}`
    }

    for (let i = 0; i < allMusic.value.data.songs.length; i++) {
      const durasi = hitungDurasi(allMusic.value.data.songs[i].duration)
      item.push({
        id: allMusic.value.data.songs[i].id,
        cover: `https://source.unsplash.com/random/${200 + i + 1}×${200 + i + 1}`,
        judul: allMusic.value.data.songs[i].title,
        tahun: allMusic.value.data.songs[i].year,
        penyanyi: allMusic.value.data.songs[i].performer,
        genre: allMusic.value.data.songs[i].genre,
        durasi,
        music: allMusic.value.data.songs[i].file,
        albumId: allMusic.value.data.songs[i].album_id
      })
    }

    return item;
  },

  set(newValue) {
    allMusic.value.data.songs = newValue
  }
})
const playingId = ref(0)
const progress = ref()
const audio = ref()
const isPlaying = ref(false)
const intervalIds = reactive([])
const formMusic = reactive({
  title: '',
  year: '',
  performer: '',
  genre: '',
  duration: '',
  albumId: '',
  file: '',
})
const fileMusic = ref()
const formAlbum = reactive({
  name: '',
  year: '',
  cover: '',
})
const fileAlbum = ref()
const openModalDeleteMusic = ref(false)
const openModalDeleteAlbum = ref(false)
const openModalEditMusic = ref(false)
const openModalEditAlbum = ref(false)
const idMusicDelete = ref('')
const idAlbumDelete = ref('')
const albumIdDisplayedSongs = ref('')
const editMusicValue = reactive({
  id: '',
  title: '',
  year: '',
  performer: '',
  genre: '',
  duration: '',
  albumId: '',
})
const editAlbumValue = reactive({
  id: '',
  name: '',
  year: ''
})

function handleLoaded() {
  progress.value.max = audio.value.duration
  progress.value.value = audio.value.currentTime
}

function playMusic() {
  isPlaying.value = true
  audio.value.play()
  const interval = setInterval(() => {
    try {
      progress.value.value = audio.value.currentTime
      if (progress.value.value == Math.floor(progress.value.max)) {
        next()
        clearInterval(interval)
      }
    } catch (error) {
      intervalIds.map((id) => {
        clearInterval(id)
      })
    }
  }, 1000);
  intervalIds.push(interval)
}

function pauseMusic() {
  isPlaying.value = false
  audio.value.pause()
  intervalIds.map((id) => {
    clearInterval(id)
  })
}

function next() {
  playingId.value++
  if (playingId.value > listMusic.value.length - 1) {
    playingId.value = 0
  }
  audio.value.src = listMusic.value[playingId.value].music
  playMusic()
}

function prev() {
  playingId.value--
  if (playingId.value < 0) {
    playingId.value = listMusic.value.length - 1
  }

  audio.value.src = listMusic.value[playingId.value].music
  playMusic()
}

function progressChange() {
  pauseMusic()
  audio.value.currentTime = progress.value.value
  playMusic()
}

async function getSongsInAlbum(albumId) {
  albumIdDisplayedSongs.value = albumId
  const {data: allSongInAlbum} = await useFetch(`http://localhost:5000/albums/${albumId}`)

  if (allSongInAlbum.value.data.album.songs.length > 0) {
    allMusic.value.data.songs = allSongInAlbum.value.data.album.songs
    playingId.value = 0
    isPlaying.value = false
    audio.value.src = listMusic.value[playingId.value].music
  } else {
    allMusic.value.data.songs = []
  }
}

async function refreshMusic() {
  if (albumIdDisplayedSongs.value) {
    const {data: musics} = await useFetch(`http://localhost:5000/albums/${albumIdDisplayedSongs.value}`)
    listMusic.value = musics.value.data.album.songs
  } else {
    const {data: musics} = await useFetch('http://localhost:5000/songs')
    listMusic.value = musics.value.data.songs
  }
  playingId.value = 0
  audio.value.src = listMusic.value[playingId.value].music
}

async function addMusic() {
  const formData = new FormData()
  formData.append('title', formMusic.title)
  formData.append('year', formMusic.year)
  formData.append('performer', formMusic.performer)
  formData.append('genre', formMusic.genre)
  formData.append('duration', formMusic.duration)
  formData.append('albumId', formMusic.albumId)
  formData.append('song', formMusic.file)

  await useFetch('http://localhost:5000/songs', {
    method: 'POST',
    body: formData
  })

  for (let item in formMusic) {
    formMusic[item] = ''
  }
  fileMusic.value.value = null
  await refreshMusic()
}

async function addAlbum() {
  const value = {
    name: formAlbum.name,
    year: formAlbum.year
  }

  const {data: responAddAlbum} = await useFetch('http://localhost:5000/albums', {
    method: 'POST',
    body: value
  })

  const formData = new FormData()
  formData.append('cover', formAlbum.cover)
  await useFetch(`http://localhost:5000/albums/${responAddAlbum.value.data.albumId}/covers`, {
    method: 'POST',
    body: formData,
  })

  for (let item in formAlbum) {
    formAlbum[item] = ''
  }
  watchValue.value++
}

function deleteMusic(id) {
  idMusicDelete.value = id
  openModalDeleteMusic.value = true
}

function deleteAlbum(id) {
  idAlbumDelete.value = id
  openModalDeleteAlbum.value = true
}

function closeModalDeleteMusic() {
  idMusicDelete.value = ''
  openModalDeleteMusic.value = false
}

function closeModalDeleteAlbum() {
  idAlbumDelete.value = ''
  openModalDeleteAlbum.value = false
}

async function deleteMusicSuccess() {
  await refreshMusic()
  idMusicDelete.value = ''
  openModalDeleteMusic.value = false
  pauseMusic()
}

async function deleteAlbumSuccess() {
  watchValue.value++
  idAlbumDelete.value = ''
  openModalDeleteAlbum.value = false
  startIndexAlbum.value = 0
  albumIdDisplayedSongs.value = ''
  await refreshMusic()
}

function editMusic(id, title, year, performer, genre, duration, albumId) {
  const splitDuration = duration.split(':')
  const totalDuration = parseInt(splitDuration[0] * 60) + parseInt(splitDuration[1])
  editMusicValue.id = id
  editMusicValue.title = title
  editMusicValue.year = year
  editMusicValue.performer = performer
  editMusicValue.genre = genre
  editMusicValue.duration = totalDuration
  editMusicValue.albumId = albumId
  openModalEditMusic.value = true
}

function editAlbum(id, name, year) {
  editAlbumValue.id = id
  editAlbumValue.name = name
  editAlbumValue.year = year
  openModalEditAlbum.value = true
}

function closeModalEditMusik() {
  for (let item in editMusicValue) {
    editMusicValue[item] = ''
  }

  openModalEditMusic.value = false
}

function closeModalEditAlbum() {
  for (let item in editAlbumValue) {
    editAlbumValue[item] = ''
  }

  openModalEditAlbum.value = false
}

async function editMusicSuccess() {
  await refreshMusic()
  closeModalEditMusik()
}

function editAlbumSuccess() {
  watchValue.value++
  closeModalEditAlbum()
}

function changeMusic(index) {
  playingId.value = index
  audio.value.src = listMusic.value[playingId.value].music
  playMusic()
}

async function getAllSongs() {
  albumIdDisplayedSongs.value = ''
  await refreshMusic()
}
onMounted(() => {
  if (listMusic.value.length > 0) {
    audio.value.src = listMusic.value[playingId.value].music
  }
})
</script>

<template>
  <div class="font-ubuntu mx-10">
    <section class="w-full h-screen flex justify-evenly items-center mb-8">
      <div class="w-[50%] h-full order-2 flex items-center">
        <img :src="image1" alt="ImageHeadphone" class="w-full object-cover object-center">
      </div>
      <div class="w-[50%] p-6 rounded-m order-1 text-slate-100">
        <h1 class="text-4xl tracking-wider font-semibold text-center mb-6 mr-9">
          Simpan Musik Favorit Anda
        </h1>
        <div class="flex mb-4">
          <div class="flex flex-col w-full gap-1">
            <label for="judulMusic" class="font-medium text-xl">Judul</label>
            <input v-model="formMusic.title" type="text" id="judulMusic" placeholder="masukan judul"
              class="w-60 placeholder:text-slate-700 px-1 h-8 rounded bg-slate-800 outline-none ring-1 ring-slate-600 focus:ring-1 focus:ring-slate-300">
          </div>
          <div class="flex flex-col w-full gap-1">
            <label for="penyanyiMusic" class="font-medium text-xl">Artis</label>
            <input v-model="formMusic.performer" type="text" id="penyanyiMusic" placeholder="masukan artis"
              class="w-60 placeholder:text-slate-700 px-1 h-8 rounded bg-slate-800 outline-none ring-1 ring-slate-600 focus:ring-1 focus:ring-slate-300">
          </div>
        </div>
        <div class="flex mb-4">
          <div class="flex flex-col w-full gap-1">
            <label for="genreMusic" class="font-medium text-xl">Genre</label>
            <input v-model="formMusic.genre" type="text" id="genreMusic" placeholder="masukan genre"
              class="w-60 placeholder:text-slate-700 px-1 h-8 rounded bg-slate-800 outline-none ring-1 ring-slate-600 focus:ring-1 focus:ring-slate-300">
          </div>
          <div class="flex flex-col w-full gap-1">
            <label for="tahunMusic" class="font-medium text-xl">Tahun</label>
            <input v-model="formMusic.year" type="text" id="tahunMusic" placeholder="masukan tahun (YYYY)"
              class="w-60 placeholder:text-slate-700 px-1 h-8 rounded bg-slate-800 outline-none ring-1 ring-slate-600 focus:ring-1 focus:ring-slate-300">
          </div>
        </div>
        <div class="flex mb-8">
          <div class="flex flex-col w-full gap-1">
            <label for="durasiMusic" class="font-medium text-xl">Durasi</label>
            <input v-model="formMusic.duration" type="text" placeholder="masukan durasi (S)" id="durasiMusic"
              class="w-60 placeholder:text-slate-700 px-1 h-8 rounded bg-slate-800 outline-none ring-1 ring-slate-600 focus:ring-1 focus:ring-slate-300">
          </div>
          <div class="flex flex-col w-full gap-1">
            <label for="albumIdMusic" class="font-medium text-xl">Album</label>
            <select id="albumIdMusic" v-model="formMusic.albumId"
              class="w-60 h-8  rounded bg-slate-800 outline-none ring-1 ring-slate-600 focus:ring-1 focus:ring-slate-300">
              <option disabled value="">Pilih satu</option>
              <option v-for="item of listAlbum.data.albums" :value="item.id">{{ item.name }}</option>
            </select>
          </div>
        </div>
        <div class="flex gap-4 items-center">
          <div>
            <label for="fileMusic" class="px-2 py-3 bg-slate-800 ring-1 ring-slate-600 rounded mr-2">
              {{ formMusic.file ? formMusic.file.name : 'Pilih file' }}
            </label>
            <button v-if="formMusic.file" @click="() => {
              fileMusic.value = null
              formMusic.file = ''
            }">
              <Icon name="mdi:close" size="24px" />
            </button>
            <input ref="fileMusic" type="file" id="fileMusic" class="hidden" accept="audio/*"
              @change="formMusic.file = fileMusic.files[0]">
          </div>
          <button @click="addMusic()"
            class="hover:scale-105 transition-transform px-2 py-[9px] bg-blue-400 rounded">Tambah</button>
        </div>
      </div>
    </section>

    <div class="w-full flex justify-center gap-4 pt-8">
      <section class="w-[35%] flex gap-4 flex-col">
        <h1 class="font-bold text-2xl text-slate-100">Sedang Diputar</h1>
        <div
          class="w-full rounded ring-[#9DB2BF] ring-1 bg-slate-800 flex items-center justify-center h-[350px] font-semibold shadow-md">
          <div v-if="allMusic.data.songs.length > 0" class="flex flex-col items-center w-full justify-center">
            <div class="w-[80px] h-[80px] rounded-2xl overflow-hidden shadow-md mb-4">
              <img :src="listMusic[playingId].cover" alt="ImageSong" class="w-full object-cover h-full object-center">
            </div>
            <h1 class="text-[#DDE6ED] text-xl tracking-wider mb-1">{{ listMusic[playingId].judul }}</h1>
            <h2 class="text-[#9DB2BF] mb-6">{{ listMusic[playingId].penyanyi }}</h2>
            <audio ref="audio" @loadedmetadata="handleLoaded">
              <source :src="listMusic[playingId].music" type="audio/mpeg">
            </audio>
            <input @change="progressChange" ref="progress" type="range" value="0" max="0" id="progress" class="mb-6">
            <div class="flex items-center gap-4">
              <div @click="playMusic" v-if="!isPlaying"
                class="w-12 h-12 rounded-full bg-[#9DB2BF] cursor-pointer hover:scale-105 transition-transform flex justify-center items-center order-2">
                <Icon name="ic:round-play-arrow" color="#DDE6ED" size="42px" />
              </div>
              <div @click="pauseMusic" v-else
                class="w-12 h-12 rounded-full bg-[#9DB2BF] cursor-pointer hover:scale-105 transition-transform flex justify-center items-center order-2">
                <Icon name="ic:round-pause" color="#DDE6ED" size="42px" />
              </div>
              <div class="order-3 hover:scale-110 cursor-pointer transition-transform" @click="next">
                <Icon name="ion:ios-skip-forward" color="#DDE6ED" size="32px" />
              </div>
              <div class="order-1 hover:scale-110 cursor-pointer transition-transform" @click="prev">
                <Icon name="ion:ios-skip-backward" color="#DDE6ED" size="32px" />
              </div>
            </div>
          </div>
        </div>
      </section>

      <section class="flex gap-4 flex-col w-[65%]">
        <h1 class="font-bold text-2xl text-slate-100">Semua Musik</h1>
        <div id="musicList"
          class="w-full cursor-pointer bg-slate-800 ring-[#9DB2BF] ring-1 h-[350px] overflow-y-auto rounded">
          <div v-if="allMusic.data.songs.length > 0" v-for="(item, index) of listMusic" :key="index"
            class="font-semibold flex py-[10px] w-full hover:bg-[#dde6ed20] rounded-md text-[#9DB2BF] hover:text-[#DDE6ED]">
            <div @click="changeMusic(index)" class="flex w-[90%] items-center justify-center">
              <span class="basis-[8%] font-semibold text-lg text-center">{{ index + 1 }}</span>
              <section class="w-[40px] h-[40px] mr-4 rounded-lg overflow-hidden shadow-lg">
                <img :src="item.cover" alt="MusicItem" class="w-full h-full object-center object-cover">
              </section>
              <h1 class="basis-[40%]">{{ item.judul }}</h1>
              <h2 class="basis-[40%]">{{ item.penyanyi }}</h2>
              <span class="basis-[12%]">{{ item.durasi }}</span>
            </div>
            <div class="w-[10%] flex gap-2 justify-center items-center">
              <button class="hover:scale-110 transition-transform">
                <Icon name="material-symbols:contract-edit-outline-rounded" class="text-[24px]"
                  @click="editMusic(item.id, item.judul, item.tahun, item.penyanyi, item.genre, item.durasi, item.albumId)" />
              </button>
              <button class="hover:scale-110 transition-transform" @click="deleteMusic(item.id)">
                <Icon name="material-symbols:delete-outline-sharp" class="text-[26px]" />
              </button>
            </div>
          </div>
        </div>
      </section>
    </div>

    <section class="w-full mt-8 mx-auto">
      <div class="w-full flex justify-between">
        <h1 class="text-3xl font-semibold text-slate-200">List Album</h1>
        <div class="flex gap-2">
          <button @click="startIndexAlbum--" v-if="startIndexAlbum > 0" class="w-9 h-9 rounded bg-slate-900">
            <Icon name="material-symbols:chevron-left" class="text-slate-300 text-3xl" />
          </button>
          <button @click="startIndexAlbum++" v-if="startIndexAlbum < listAlbum.data.albums.length - 4"
            class="w-9 h-9 rounded bg-slate-900">
            <Icon name="material-symbols:chevron-right" class="text-slate-300 text-3xl" />
          </button>
        </div>
      </div>
      <div class="w-full flex justify-center gap-4 mt-4">
        <div class="w-[20%] cursor-pointer transition-transform hover:scale-105 flex flex-col relative">
          <div @click="getAllSongs()">
            <div class="w-full h-[200px] overflow-hidden rounded-lg shadow-xl mb-4">
              <img :src="image3" alt="ImageAlbum" class="object-cover w-full h-full object-center">
            </div>
            <h1 class="font-semibold text-xl mb-1 text-[#DDE6ED]">
              Semua Musik
            </h1>
            <span class="font-semibold text-lg text-[#9DB2BF]">
              2023
            </span>
          </div>
        </div>

        <div v-for="item of newAlbums" :key="item.id"
          class="w-[20%] cursor-pointer transition-transform hover:scale-105 flex flex-col relative">
          <div @click="getSongsInAlbum(item.id)">
            <div class="w-full h-[200px] overflow-hidden rounded-lg shadow-xl mb-4">
              <img :src="item.cover" alt="ImageAlbum" class="object-cover w-full h-full object-center">
            </div>
            <h1 class="font-semibold text-xl mb-1 text-[#DDE6ED]">
              {{ item.name }}
            </h1>
            <span class="font-semibold text-lg text-[#9DB2BF]">
              {{ item.year }}
            </span>
          </div>
          <div class="w-10 h-10 absolute right-0 -top-1 z-30">
            <button class="px-3 py-3" @click="openMenuAlbums(item.id)">
              <Icon :name="menuAlbums[item.id] ? 'material-symbols:close' : 'mdi:dots-vertical'" size="24px"
                class="text-slate-50" />
            </button>
          </div>
          <div v-if="menuAlbums[item.id]"
            class="w-28 h-14 justify-center items-center bg-[#526D82] text-slate-50 absolute flex flex-col right-8 rounded top-3 z-50">
            <button class="w-full h-full hover:bg-[#63859f] rounded"
              @click="editAlbum(item.id, item.name, item.year)">Edit</button>
            <button class="w-full h-full hover:bg-[#63859f] rounded" @click="deleteAlbum(item.id)">Hapus</button>
          </div>
        </div>
      </div>
    </section>

    <div class="mt-12 flex justify-around ">
      <div class="w-[30%] p-6 rounded-md order-2 mt-8 text-slate-100">
        <h1 class="text-3xl text-center font-semibold">Tambah Album</h1>
        <label for="namaAlbum" class="block font-medium leading-6">Nama</label>
        <div class="mt-2">
          <input type="text" v-model="formAlbum.name" id="namaAlbum" placeholder="masukan nama"
            class="block w-full placeholder:text-[#3f5679bb] rounded-md border-0 p-1.5 shadow-sm ring-1 ring-inset ring-[#9DB2BF] text-[#DDE6ED]  focus:outline-none focus:ring-[#DDE6ED] bg-[#27374D]">
        </div>

        <label for="tahunAlbum" class="block mt-4 font-medium leading-6">Tahun</label>
        <div class="mt-2">
          <input v-model="formAlbum.year" type="text" id="tahunAlbum" placeholder="masukan tahun (YYYY)"
            class="block w-full rounded-md border-0 placeholder:text-[#3f5679bb] p-1.5 shadow-sm ring-1 ring-inset ring-[#9DB2BF] text-[#DDE6ED] focus:outline-none focus:ring-[#DDE6ED] bg-[#27374D]">
        </div>

        <div class="flex gap-4 items-center mt-6">
          <div>
            <label for="fileAlbum" class="px-2 py-3 ring-1 ring-[#9DB2BF] rounded mr-2">
              {{ formAlbum.cover ? formAlbum.cover.name : 'Pilih file' }}
            </label>
            <button v-if="formAlbum.cover" @click="() => {
              fileAlbum.value = null
              formAlbum.cover = ''
            }">
              <Icon name="mdi:close" size="24px" />
            </button>
            <input ref="fileAlbum" type="file" id="fileAlbum" class="hidden" accept="image/*" @change="() => {
              formAlbum.cover = fileAlbum.files[0]
            }">
          </div>
          <button @click="addAlbum()"
            class="px-2 py-[9px] bg-blue-400 rounded hover:scale-105 transition-transform">Tambah</button>
        </div>
      </div>
      <div class="w-[45%] h-full flex items-center order-1">
        <img :src="image2" alt="ImageHeadphone" class="w-full object-cover object-center">
      </div>
    </div>
  </div>
  <ModalDelete v-if="openModalDeleteMusic" :id="idMusicDelete" entitas="songs" @cancel-delete="closeModalDeleteMusic"
    @success-delete="deleteMusicSuccess">
    <h1 class="text-slate-50 font-semibold text-xl">Hapus musik</h1>
    <p class="font-light text-slate-50">Apakah anda yakin ingin menghapus musik ini? musik akan di hapus
      permanent dan tidak dapat di kembalikan</p>
  </ModalDelete>

  <ModalDelete v-if="openModalDeleteAlbum" :id="idAlbumDelete" entitas="albums" @cancel-delete="closeModalDeleteAlbum"
    @success-delete="deleteAlbumSuccess">
    <h1 class="text-slate-50 font-semibold text-xl">Hapus album</h1>
    <p class="font-light text-slate-50">Apakah anda yakin ingin menghapus album ini? album akan di hapus
      permanent dan tidak dapat di kembalikan</p>
  </ModalDelete>

  <ModalEdit v-if="openModalEditMusic" @success-edit="editMusicSuccess" @cancel-edit="closeModalEditMusik"
    :edit-value="editMusicValue" entitas="songs">
    <h1 class="text-slate-50 font-semibold text-3xl mb-4">Edit Musik</h1>
    <div class="flex mb-4 w-full gap-6">
      <div class="flex flex-col gap-1 w-1/2">
        <label for="editJudulMusic" class="font-light text-lg text-slate-50">Judul</label>
        <input v-model="editMusicValue.title" type="text" id="editJudulMusic" placeholder="masukan judul"
          class="w-full bg-[#9DB2BF] placeholder:text-[#9db2bfc5] px-1 h-8 rounded outline-none ring-1 ring-slate-50 text-slate-50 focus:ring-1 focus:ring-slate-300">
      </div>
      <div class="flex flex-col gap-1 w-1/2">
        <label for="editPenyanyiMusic" class="font-light text-lg text-slate-50">Artis</label>
        <input v-model="editMusicValue.performer" type="text" id="editPenyanyiMusic" placeholder="masukan artis"
          class="w-full bg-[#9DB2BF] placeholder:text-[#9db2bfc5] px-1 h-8 rounded outline-none ring-1 ring-slate-50 text-slate-50 focus:ring-1 focus:ring-slate-300">
      </div>
    </div>
    <div class="flex w-full gap-6">
      <div class="flex flex-col w-1/2 gap-1">
        <label for="editGenreMusic" class="font-light text-lg text-slate-50">Genre</label>
        <input v-model="editMusicValue.genre" type="text" id="editGenreMusic" placeholder="masukan genre"
          class="w-full bg-[#9DB2BF] placeholder:text-[#9db2bfc5] px-1 h-8 rounded outline-none ring-1 ring-slate-50 text-slate-50 focus:ring-1 focus:ring-slate-300">
      </div>
      <div class="flex w-1/2 gap-3">
        <div class="flex flex-col w-full gap-1">
          <label for="editTahunMusic" class="font-light text-lg text-slate-50">Tahun</label>
          <input v-model="editMusicValue.year" type="text" id="editTahunMusic" placeholder="(YYYY)"
            class="w-full bg-[#9DB2BF] placeholder:text-[#9db2bfc5] px-1 h-8 rounded outline-none ring-1 ring-slate-50 text-slate-50 focus:ring-1 focus:ring-slate-300">
        </div>
        <div class="flex flex-col w-full gap-1">
          <label for="editDurasiMusic" class="font-light text-lg text-slate-50">Durasi</label>
          <input v-model="editMusicValue.duration" type="text" placeholder="(S)" id="editDurasiMusic"
            class="w-full bg-[#9DB2BF] placeholder:text-[#9db2bfc5] px-1 h-8 rounded outline-none ring-1 ring-slate-50 text-slate-50 focus:ring-1 focus:ring-slate-300">
        </div>
      </div>
    </div>
  </ModalEdit>

  <ModalEdit v-if="openModalEditAlbum" @success-edit="editAlbumSuccess" @cancel-edit="closeModalEditAlbum"
    :edit-value="editAlbumValue" entitas="albums">
    <h1 class="text-slate-50 font-semibold text-3xl mb-4">Edit Album</h1>
    <div class="flex flex-col w-full gap-1">
      <label for="editNamaAlbum" class="font-light text-lg text-slate-50">Nama</label>
      <input v-model="editAlbumValue.name" type="text" id="editNamaAlbum" placeholder="masukan nama"
        class="w-full bg-[#9DB2BF] placeholder:text-[#9db2bfc5] px-1 h-8 rounded outline-none ring-1 ring-slate-50 text-slate-50 focus:ring-1 focus:ring-slate-300">
    </div>
    <div class="flex flex-col w-full gap-1">
      <label for="editTahunAlbum" class="font-light text-lg text-slate-50">Tahun</label>
      <input v-model="editAlbumValue.year" type="text" id="editTahunAlbum" placeholder="(YYYY)"
        class="w-full bg-[#9DB2BF] placeholder:text-[#9db2bfc5] px-1 h-8 rounded outline-none ring-1 ring-slate-50 text-slate-50 focus:ring-1 focus:ring-slate-300">
    </div>
  </ModalEdit>
</template>

<style>
body {
  background: #27374D;
}

body::-webkit-scrollbar {
  width: 0px;
}

#musicList::-webkit-scrollbar {
  width: 8px;
}

#musicList::-webkit-scrollbar-thumb {
  background-color: #9DB2BF;
  border-radius: 10px;
}

#progress {
  -webkit-appearance: none;
  width: 80%;
  height: 6px;
  background: #9DB2BF;
  border-radius: 4px;
  cursor: pointer;
}

#progress::-webkit-slider-thumb {
  -webkit-appearance: none;
  background: #DDE6ED;
  width: 12px;
  height: 12px;
  border-radius: 50%;
}
</style>