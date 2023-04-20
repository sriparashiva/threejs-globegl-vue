<script setup>
  import Globe from 'globe.gl'
  import * as topojson from 'topojson-client'
  import { onMounted, ref, computed } from 'vue'

  let world

  const locationClicked = ref(false)
  const mapData = ref({ features: [] })
  const searchQuery = ref('')
  const currentPage = ref(1)
  const globeReady = ref(false)
  const globeElement = ref(null)

  const getMapData = async () => {
    const data = await fetch(
      `https://api.jsonbin.io/v3/b/6440debdace6f33a220f0a12/latest`,
      // `https://api.jsonbin.io/v3/b/6264215e019db46796905e1c/latest`,
      {
        method: 'GET',
        headers: {
          'X-Master-key':
            '$2b$10$nJssfrDDCqG11fIJZ7JqXuK/hlqctdaOVJS7FHP4.05kMqL0qCdKK',
        },
      }
    )
    return data.json()
  }

  const getTopoData = async () => {
    const data = await fetch(
      `https://api.jsonbin.io/v3/b/626421fd38be296761f6e03c/latest`,
      {
        method: 'GET',
        headers: {
          'X-Master-key':
            '$2b$10$nJssfrDDCqG11fIJZ7JqXuK/hlqctdaOVJS7FHP4.05kMqL0qCdKK',
        },
      }
    )
    return data.json()
  }

  const closePopups = () => {
    const popups = document.getElementsByClassName('marker__popup')
    if (popups.length > 0) {
      for (let i = 0; i < popups.length; i++) {
        popups[i].remove()
      }
      locationClicked.value = false
    }
  }

  const createPopup = (location, el) => {
    let popup = document.createElement('div')
    popup.classList.add('marker__popup', 'shadow-xl', 'pointer-events-none')
    popup.innerHTML = `<div class="closeIcon z-20 cursor-pointer pointer-events-auto"><svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4" viewBox="0 0 20 20" fill="currentColor">
    <path fill-rule="evenodd" d="M10 18a8 8 0 100-16 8 8 0 000 16zM8.707 7.293a1 1 0 00-1.414 1.414L8.586 10l-1.293 1.293a1 1 0 101.414 1.414L10 11.414l1.293 1.293a1 1 0 001.414-1.414L11.414 10l1.293-1.293a1 1 0 00-1.414-1.414L10 8.586 8.707 7.293z" clip-rule="evenodd" />
  </svg></div><h2 class="font-medium text-sm leading-tight">${location.title}</h2><p class="text-xs opacity-50">${location.type}</p><p class="text-xs text-[#7A4230]">${location.state}, ${location.country}</p>`
    el.appendChild(popup)
  }

  const initializeClosePopups = () => {
    let closeEl = document.getElementsByClassName('marker__popup')[0]
    closeEl.onclick = (event) => {
      event.stopPropagation()
      closePopups()
      world.controls().autoRotate = true
    }
  }

  const setPOV = (location) => {
    world.pointOfView(
      {
        lat: location.lat,
        lng: location.lng,
        // altitude: 1.5,
      },
      750
    )
    world.controls().autoRotate = false
  }

  const clickLocation = (location, el) => {
    closePopups()
    createPopup(location, el)
    initializeClosePopups()
    locationClicked.value = true
    setPOV(location)
  }

  const listItemClick = (clickedLocation) => {
    const location = {
      id: clickedLocation.id,
      lat: clickedLocation.geometry.coordinates[1],
      lng: clickedLocation.geometry.coordinates[0],
      title: clickedLocation.properties.Title,
      city: clickedLocation.properties.City,
      state: clickedLocation.properties.State,
      country: clickedLocation.properties.Country,
      type: clickedLocation.properties.Type,
    }
    closePopups()
    const marker = document.getElementById(`marker_${location.id}`)
    createPopup(location, marker)
    initializeClosePopups()
    locationClicked.value = true
    setPOV(location)
  }

  const filteredList = computed(() => {
    if (searchQuery.value.length > 2) {
      currentPage.value = 1
      const titles = mapData.value.features.filter((feature) =>
        feature.properties.Title.toLowerCase().includes(
          searchQuery.value.toLowerCase().trim()
        )
      )
      const countries = mapData.value.features.filter((feature) =>
        feature.properties.Country.toLowerCase().includes(
          searchQuery.value.toLowerCase().trim()
        )
      )
      const state = mapData.value.features.filter((feature) =>
        feature.properties.State.toLowerCase().includes(
          searchQuery.value.toLowerCase().trim()
        )
      )
      const city = mapData.value.features.filter((feature) =>
        feature.properties.City.toLowerCase().includes(
          searchQuery.value.toLowerCase().trim()
        )
      )
      return [...new Set([...titles, ...countries, ...state, ...city])]
    } else {
      return mapData.value.features
    }
  })

  const totalPages = computed(() => {
    let pages = Math.floor(filteredList.value.length / 8)
    pages += filteredList.value.length % 8 > 0 ? 1 : 0
    return pages
  })

  const paginatedList = computed(() => {
    return filteredList.value.slice(
      (currentPage.value - 1) * 8,
      currentPage.value * 8 - 1
    )
  })

  const initializeGlobe = (gData, landTopo) => {
    const markerSvg = `<svg class="markerSvg" xmlns="http://www.w3.org/2000/svg" viewBox="-4 0 36 36"><path fill="none" d="M0 0h24v24H0z"/><path fill="currentColor" d="M18.364 17.364L12 23.728l-6.364-6.364a9 9 0 1 1 12.728 0zM12 13a2 2 0 1 0 0-4 2 2 0 0 0 0 4z"/></svg>`

    world = Globe()(globeElement.value)
      .width([document.getElementsByClassName('globeContent')[0].offsetWidth])
      .height([document.getElementsByClassName('globeContent')[0].offsetWidth])
      .backgroundColor('rgba(0,0,0,0)')
      .globeImageUrl('https://i.imgur.com/fGPqeDC.png')
      .showAtmosphere(false)
      .enablePointerInteraction(false)
      .pointOfView({ altitude: 1.8 })
      .htmlElementsData(gData)
      .htmlElement((d) => {
        const el = document.createElement('div')
        el.innerHTML = markerSvg
        el.style.color = d.color
        el.style.width = `${d.size}px`
        el.classList.add('mapMarker')
        el.style['pointer-events'] = 'auto'
        el.style.cursor = 'pointer'
        el.id = `marker_${d.id}`
        el.onclick = () => clickLocation(d, el)
        el.addEventListener(
          'mouseenter',
          (e) => (world.controls().autoRotate = false)
        )
        el.addEventListener('mouseleave', (e) => {
          if (locationClicked.value == false) world.controls().autoRotate = true
        })
        return el
      })

    world.controls().autoRotate = true
    world.controls().enableZoom = false
    world.controls().enablePan = false
    world.controls().autoRotateSpeed = 200

    setTimeout(() => {
      world.controls().autoRotateSpeed = 1.2
      globeReady.value = true
    }, 1000)

    world
      .polygonsData(topojson.feature(landTopo, landTopo.objects.land).features)
      .polygonCapColor((feat) => 'rgba(58, 52, 149, 1)')
      .polygonSideColor(() => 'rgba(0,0,0,0)')

    window.addEventListener('resize', (event) => {
      world.width([
        document.getElementsByClassName('globeContent')[0].offsetWidth,
      ])
      world.height([
        document.getElementsByClassName('globeContent')[0].offsetWidth,
      ])
    })
  }

  onMounted(async () => {
    let { record } = await getMapData()
    mapData.value = record
    const { record: landTopo } = await getTopoData()
    const gData = mapData.value.features.map((feature) => ({
      id: feature.id,
      lat: feature.geometry.coordinates[1],
      lng: feature.geometry.coordinates[0],
      title: feature.properties.Title,
      city: feature.properties.City,
      state: feature.properties.State,
      country: feature.properties.Country,
      type: feature.properties.Type,
      size: 24,
      color: '#EC4F11',
    }))
    initializeGlobe(gData, landTopo)
  })
</script>

<template>
  <main>
    <div
      v-show="!globeReady"
      class="w-screen h-screen flex justify-center items-center"
    >
      <div class="Loader loader mx-auto"></div>
    </div>
    <div
      class="mainContent w-full flex flex-col md:flex-row justify-center md:py-6"
      :class="globeReady ? 'showGlobe' : ''"
    >
      <div
        class="globeContent w-full md:w-1/2 basis-full md:basis-1/2 flex-grow-0 flex-shrink"
      >
        <div ref="globeElement" id="globeViz"></div>
      </div>
      <div
        class="listContent basis-full md:(basis-1/2) flex-shrink-0 flex-grow flex flex-col justify-center"
      >
        <div class="searchInput px-4 pb-2 relative">
          <button
            v-if="searchQuery.length > 0"
            class="closeIcon absolute right-0 z-2 mt-1 mr-3 opacity-50"
            @click="searchQuery = ''"
          >
            <svg
              xmlns="http://www.w3.org/2000/svg"
              class="h-5 w-5 opacity-50 transition hover:(opacity-100 scale-110)"
              viewBox="0 0 20 20"
              fill="currentColor"
            >
              <path
                fill-rule="evenodd"
                d="M10 18a8 8 0 100-16 8 8 0 000 16zM8.707 7.293a1 1 0 00-1.414 1.414L8.586 10l-1.293 1.293a1 1 0 101.414 1.414L10 11.414l1.293 1.293a1 1 0 001.414-1.414L11.414 10l1.293-1.293a1 1 0 00-1.414-1.414L10 8.586 8.707 7.293z"
                clip-rule="evenodd"
              />
            </svg>
          </button>
          <input
            type="text"
            name="searchQuery"
            autocomplete="false"
            id="searchQuery"
            placeholder="Search for a location near you..."
            class="outline-none pb-1 border-b-1 border-gray-400 border-opacity-40 text-sm w-full bg-transparent"
            v-model="searchQuery"
          />
        </div>
        <div class="listContainer overflow-y-scroll">
          <ul>
            <li
              v-for="item in paginatedList"
              :key="item.id"
              class="px-4 py-2 border-b-1 border-gray-400 border-opacity-15 cursor-pointer"
              @click="listItemClick(item)"
            >
              <h3 class="text-base font-medium text-[#332e77]">
                {{ item.properties.Title }}
              </h3>
              <p class="text-xs opacity-50">{{ item.properties.Type }}</p>
              <p class="text-xs text-[#7A4230]">
                {{ item.properties.State }}, {{ item.properties.Country }}
              </p>
            </li>
          </ul>
          <div class="pagination flex justify-between px-4 py-2 text-[#332e77]">
            <button
              class="buttonPrev text-sm transition mr-auto hover:text-[#ec4f11]"
              :disabled="currentPage <= 1"
              @click="currentPage--"
            >
              PREVIOUS
            </button>
            <button
              class="buttonNext text-sm transition ml-auto hover:text-[#ec4f11]"
              :disabled="currentPage >= totalPages"
              @click="currentPage++"
            >
              NEXT
            </button>
          </div>
        </div>
      </div>
    </div>
  </main>
</template>

<style lang="scss">
  .mapMarker {
    svg,
    img {
      transition: all 0.3s ease-in-out;
    }
    svg.markerSvg,
    img {
      &:hover {
        transform: scale(1.4);
      }
    }
  }
  .marker__popup {
    position: absolute;
    bottom: 0;
    left: 0;
    padding: 0.4rem 0.7rem;
    background-color: white;
    color: #221f4b;
    width: 10rem;
    word-break: normal;
    white-space: normal;
    margin-left: 0.8rem;
    margin-bottom: 0.8rem;
    .closeIcon {
      position: absolute;
      top: 0;
      right: 0;
      margin: 0.1rem;
      opacity: 0.6;
      transition: all 0.2s ease-in-out;
      &:hover {
        opacity: 1;
        scale: (1.02);
      }
    }
  }
  .listContainer {
    max-height: inherit;
  }
  .pagination button[disabled] {
    display: none;
  }
  .mainContent {
    opacity: 0;
    transition: opacity 0.3s ease;
  }
  .showGlobe {
    opacity: 1;
  }
  .mainContent .pagination button {
    border: none;
    background: none;
    color: #332e77;
    cursor: pointer;
    transition: color 0.3s ease-in-out;
    font-size: 12px;
    letter-spacing: 2px;
  }
  .mainContent .pagination button:hover {
    color: #ec4f11;
  }
  .loader {
    animation: rotate 1.3s infinite;
    height: 50px;
    width: 50px;
  }

  .loader:before,
  .loader:after {
    border-radius: 50%;
    content: '';
    display: block;
    height: 20px;
    width: 20px;
  }
  .loader:before {
    animation: ball1 1.3s infinite;
    background-color: #ec4f11;
    box-shadow: 30px 0 0 #332e77;
    margin-bottom: 10px;
  }
  .loader:after {
    animation: ball2 1.3s infinite;
    background-color: #332e77;
    box-shadow: 30px 0 0 #97bf0d;
  }

  @keyframes rotate {
    0% {
      -webkit-transform: rotate(0deg) scale(0.8);
      -moz-transform: rotate(0deg) scale(0.8);
    }
    50% {
      -webkit-transform: rotate(360deg) scale(1.2);
      -moz-transform: rotate(360deg) scale(1.2);
    }
    100% {
      -webkit-transform: rotate(720deg) scale(0.8);
      -moz-transform: rotate(720deg) scale(0.8);
    }
  }

  @keyframes ball1 {
    0% {
      box-shadow: 30px 0 0 #332e77;
    }
    50% {
      box-shadow: 0 0 0 #332e77;
      margin-bottom: 0;
      -webkit-transform: translate(15px, 15px);
      -moz-transform: translate(15px, 15px);
    }
    100% {
      box-shadow: 30px 0 0 #332e77;
      margin-bottom: 10px;
    }
  }

  @keyframes ball2 {
    0% {
      box-shadow: 30px 0 0 #ec4f11;
    }
    50% {
      box-shadow: 0 0 0 #ec4f11;
      margin-top: -20px;
      -webkit-transform: translate(15px, 15px);
      -moz-transform: translate(15px, 15px);
    }
    100% {
      box-shadow: 30px 0 0 #ec4f11;
      margin-top: 0;
    }
  }
</style>
