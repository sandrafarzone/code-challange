<template>
  <v-container class="fill-height">
    <v-responsive class="align-center text-center fill-height">
      <span class="text-h4 font-weight-bold">Traffic Info</span>
      <h1 class="text-subtitle-1">
        Get local information about the current traffic situation
      </h1>

      <div class="py-8" />

      <v-row class="d-flex align-center justify-center">
        <v-col cols="auto">
          <v-btn
            color="primary"
            variant="outlined"
            prepend-icon="mdi-map-marker"
            @click="useCurrentLocationClicked"
            :loading="isFetchingLocation"
          >
            Use current location
          </v-btn>
          <v-select
            v-model="selectedArea"
            class="mt-5"
            label="Area"
            clearable
            :items="selectableAreas"
            variant="outlined"
            :disabled="isFetchingLocation"
            @click:clear="fetchedMessages = []"
          ></v-select>
        </v-col>
      </v-row>
      <template v-if="!isFetchingLocation">
        <h1
          class="text-subtitle-1 mb-5 mt-10 font-weight-bold"
          v-if="fetchedMessages.length > 0"
        >
          Traffic Messages for {{ selectedArea }}:
        </h1>
        <h1 class="text-subtitle-1 mb-5 mt-10 font-weight-bold" v-else>
          No traffic messages
        </h1>
        <v-row class="d-flex align-center justify-center">
          <v-card
            class="mb-5"
            max-width="500"
            v-for="message in fetchedMessages"
            variant="outlined"
            :title="message.title"
            :subtitle="message.exactLocation"
          >
            <v-card-text>
              <p>Description: {{ message.description }}</p>
              <p>Priority: {{ message.priority }}</p>
              <p>
                Category: {{ message.subCategory }} ({{ message.category }})
              </p>
            </v-card-text>
          </v-card>
        </v-row>
      </template>
    </v-responsive>
  </v-container>
</template>

<script setup>
import { onMounted, ref, watch } from "vue";
import axios from "axios";

const xmlParser = new DOMParser();
const fetchedMessages = ref([]);
const selectableAreas = ref([]);
const selectedArea = ref("");

const isFetchingLocation = ref(false);

function useCurrentLocationClicked() {
  isFetchingLocation.value = true;
  navigator.geolocation.getCurrentPosition(
    (position) => {
      getArea(position);
      isFetchingLocation.value = false;
    },
    (error) => {
      console.log(error);
      isFetchingLocation.value = false;
    }
  );
}

function convertResponseToList(response) {
  fetchedMessages.value = [];

  for (let i = 0; i < response.length; i++) {
    let msg = response[i];
    let attributes = msg.attributes;
    let msgObject = {
      id: attributes["id"].nodeValue,
      priority: attributes["priority"].nodeValue,
      createdDate: getValue(msg, "createddate"),
      title: getValue(msg, "title"),
      exactLocation: getValue(msg, "exactlocation"),
      description: getValue(msg, "description"),
      latitude: getValue(msg, "latitude"),
      longitude: getValue(msg, "longitude"),
      category: getValue(msg, "category"),
      subCategory: getValue(msg, "subcategory"),
    };
    fetchedMessages.value.push(msgObject);
  }
}

function getValue(msg, tagName) {
  try {
    return msg.getElementsByTagName(tagName)[0].childNodes[0].nodeValue;
  } catch {
    return "";
  }
}

async function getArea(position) {
  await axios
    .get("http://api.sr.se/api/v2/traffic/areas", {
      params: {
        latitude: position.coords.latitude,
        longitude: position.coords.longitude,
      },
    })
    .then((response) => {
      const xmlDoc = xmlParser.parseFromString(
        response.data.trim(),
        "text/xml"
      );
      selectedArea.value =
        xmlDoc.getElementsByTagName("area")[0].attributes["name"].nodeValue;
    });
}

async function getAllAreas() {
  await axios
    .get("http://api.sr.se/api/v2/traffic/areas?pagination=false")
    .then((response) => {
      const xmlDoc = xmlParser.parseFromString(
        response.data.trim(),
        "text/xml"
      );

      const xmlAreas = xmlDoc.getElementsByTagName("area");
      for (let i = 0; i < xmlAreas.length; i++) {
        let areaName = xmlAreas[i].attributes["name"].nodeValue;
        selectableAreas.value.push(areaName);
      }
    });
}

async function getTraffic() {
  await axios
    .get("http://api.sr.se/api/v2/traffic/messages?pagination=false", {
      params: {
        trafficareaname: selectedArea.value,
        date: new Date().getDate(),
      },
    })
    .then((response) => {
      const xmlDoc = xmlParser.parseFromString(
        response.data.trim(),
        "text/xml"
      );

      convertResponseToList(xmlDoc.getElementsByTagName("message"));
    })
    .catch((error) => {
      fetchedMessages.value = [];
      console.error(error);
    });
}

watch(selectedArea, () => {
  if (selectedArea.value !== null) {
    getTraffic();
  }
});

onMounted(() => {
  getAllAreas();
});
</script>
