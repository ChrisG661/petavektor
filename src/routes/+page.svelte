<script>
  import { onMount, onDestroy } from "svelte";
  import { browser } from "$app/environment";
  import {
    Heading,
    Input,
    Label,
    Helper,
    ButtonGroup,
    Button,
    Range,
  } from "flowbite-svelte";

  let mapElement;
  let map;
  let drawRoute;
  let calculateRoute;
  let currentRoute;
  let drawVector;
  let threshold = 0.00003;
  // @ts-ignore
  let savedRoutes = {
    /*
    rsup: {
      name: "",
      start: [0, 0],
      destination: [0, 0],
      geometry: [],
      simplified: [],
      tolerance: 0,
      vectors: [
        {
          name: "",
          start: [],
          start_name: "",
          end: [],
          end_name: "",
          position: [],
          distanceX: 0,
          distanceY: 0,
          distance: 0,
          bearing: 0,
        },
      ],
    },*/
  };

  function numberToLetters(num) {
    var s = "",
      t;

    while (num > 0) {
      t = (num - 1) % 26;
      s = String.fromCharCode(65 + t) + s;
      num = ((num - t) / 26) | 0;
    }
    return s || undefined;
  }
  function swapToLatLong(coordinates) {
    coordinates.forEach((coordinate) => {
      [coordinate[0], coordinate[1]] = [coordinate[1], coordinate[0]];
    });
    return coordinates;
  }

  function swapToLongLat(coordinates) {
    coordinates.forEach((coordinate) => {
      [coordinate[0], coordinate[1]] = [coordinate[1], coordinate[0]];
    });
    return coordinates;
  }

  onMount(async () => {
    if (browser) {
      const L = await import("leaflet");
      const turf = await import("@turf/turf");
      const arrowheads = await import("leaflet-arrowheads");

      map = L.map(mapElement).setView([1.438654, 124.790774], 13);

      L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {
        attribution:
          '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
      }).addTo(map);

      L.marker([1.437757, 124.790186]).addTo(map).bindPopup("Rumah");
      //.openPopup();

      async function getRouteGeometry(start, destination, full) {
        const geometry = await fetch(
          `https://router.project-osrm.org/route/v1/driving/${start[1]},${
            start[0]
          };${destination[1]},${
            destination[0]
          }?generate_hints=false&alternatives=false&geometries=geojson&overview=${
            full ? "full" : "simplified"
          }&steps=false`,
          {
            method: "GET",
            cache: "force-cache",
          }
        )
          .then((response) => {
            return response.json();
          })
          .then((data) => {
            const coordinates = swapToLatLong(
              data.routes[0].geometry.coordinates
            );
            return coordinates;
          })
          .catch((error) => {
            console.error(error);
          });
        return geometry;
      }

      let vectorComponents = (start, end) => {
        let x = end[1] - start[1];
        let y = end[0] - start[0];
        let position = [x, y];
        let distance = turf.distance(start, end, { units: "meters" });
        let bearing = turf.bearing(start, end);
        let distanceX = turf.distance([0, 0], [x, 0], { units: "meters" });
        let distanceY = turf.distance([0, 0], [0, y], { units: "meters" });
        return {
          start,
          end,
          position,
          distanceX,
          distanceY,
          distance,
          bearing,
        };
      };

      calculateRoute = async (start, destination, tolerance) => {
        return await getRouteGeometry(start, destination, true).then(
          (coordinates) => {
            console.log(coordinates);
            let geometry = coordinates;
            let simplified = turf.simplify(turf.lineString(coordinates), {
              tolerance: tolerance,
            }).geometry.coordinates;

            let vectors = [];
            for (let i = 0; i < simplified.length - 1; i++) {
              let start_name = numberToLetters(i + 1);
              let name = start_name?.toLowerCase();
              let end_name = numberToLetters(i + 2);
              let vector = {
                ...vectorComponents(simplified[i], simplified[i + 1]),
                name,
                start_name,
                end_name,
              };
              //console.log(vector);

              vectors.push(vector);
            }
            return {
              start,
              destination,
              geometry,
              simplified,
              tolerance,
              vectors,
            };
          }
        );
      };

      drawRoute = async (start, destination, full = false) => {
        await getRouteGeometry(start, destination, full).then((coordinates) => {
          L.polyline(coordinates, {
            weight: 4,
            className: "stroke-gray-600 opacity-75",
          }).addTo(map);
          let simplified = turf.simplify(turf.lineString(coordinates), {
            tolerance: threshold,
          });
          L.polyline(simplified.geometry.coordinates, {
            className: "stroke-blue-400",
          }).addTo(map);
          //calculateRoute(start, destination, threshold);
        });
      };

      drawVector = (vector) => {
        console.dir(vector);
        let start = vector.start;
        let end = vector.end;
        let position = vector.position;
        let distance = vector.distance;
        let distanceX = vector.distanceX;
        let distanceY = vector.distanceY;
        let bearing = vector.bearing;
        let name = vector.name;
        let start_name = vector.start_name;
        let end_name = vector.end_name;

        let vectorLine = L.polyline([start, end], { color: "#2563eb" })
          .addTo(map)
          .arrowheads({ size: "12px", fill: true, yawn: 40 });
        //let startPoint = L.circleMarker(start, { color: "#1d4ed8" }).addTo(map);
        //let endPoint = L.circleMarker(end, { color: "#1d4ed8" }).addTo(map);
        map.fitBounds(vectorLine.getBounds());
      };

      function drawRoutePoint(coordinate) {
        L.circle(coordinate, { icon: L.divIcon() }).addTo(map);
      }
      //drawRoute([1.437533, 124.79023], [1.46146, 124.837412], true);
      //drawRoutePoint([1.46146, 124.837412]);
    }
  });

  async function loadRoute(route) {
    currentRoute = route;
    currentVectorIndex = 0;
    currentVector = {};
    let start = [1.437757, 124.790186];
    let destination = [0, 0];
    let name = "";
    switch (route) {
      case "rsup": {
        destination = [1.455297, 124.807755];
        name = "RSUP Prof. Kandou";
        break;
      }
      case "sdh": {
        destination = [1.46146, 124.837412];
        name = "Sekolah Dian Harapan Manado";
        break;
      }
      case "linow": {
        destination = [1.272947, 124.822804];
        name = "Danau Linow";
        break;
      }
    }

    if (browser) {
      let data = await calculateRoute(start, destination, threshold);
      savedRoutes[route] = { name, ...data };
      drawRoute(start, destination, true);
      drawVector(savedRoutes[route].vectors[currentVectorIndex]);
    }
    console.log(route, currentRoute);
  }

  onDestroy(async () => {
    if (map) {
      console.log("Unloading Leaflet map.");
      map.remove();
    }
  });

  let currentVector = {
    name: "",
    start: [0, 0],
    start_name: "",
    end: [0, 0],
    end_name: "",
    position: [0, 0],
    distanceX: 0,
    distanceY: 0,
    distance: 0,
    bearing: 0,
  };
  let currentVectorIndex = 0;

  function viewVector(index) {
    console.log(index);
    let vector = savedRoutes[currentRoute]?.vectors[index];
    console.log(vector);
    currentVector = vector;
    currentVectorIndex = index;
    drawVector(vector);
  }
</script>

<main class="h-screen w-screen">
  <div class="flex flex-row">
    <div class="w-[28rem] border-r-2 flex flex-col gap-y-4 p-4 border-gray-200">
      <Heading tag="h3">Peta Vektor</Heading>
      <ButtonGroup class="mx-auto">
        <Button id="rsup" on:click={() => loadRoute("rsup")}>RSUP</Button>
        <Button id="sdh" on:click={() => loadRoute("sdh")}>SDH</Button>
        <Button id="linow" on:click={() => loadRoute("linow")}>Linow</Button>
      </ButtonGroup>
      <div>
        <Label for="start-point" class="mb-2">Titik awal</Label>
        <Input
          type="text"
          id="start-point"
          size="sm"
          placeholder="Rumah Christopher"
          required
        />
      </div>
      <div>
        <Label for="destination-point" class="mb-2">Titik akhir</Label>
        <Input
          type="text"
          id="destination-point"
          size="sm"
          placeholder=""
          required
        />
      </div>
      <div class="gap-y-2">
        <Heading tag="h5" class="mb-2">Vektor {currentVector?.name}</Heading>

        <div class="flex flex-row gap-x-2 gap-y-2">
          <div>
            <label for="start-vector" class="mb-2 text-sm font-semibold"
              >Pangkal</label
            >
            <Input
              type="text"
              id="start-vector"
              size="sm"
              placeholder="0"
              required
              disabled
            />
          </div>
          <div>
            <label for="end-vector" class="mb-2 text-sm font-semibold"
              >Ujung</label
            >
            <Input
              type="text"
              id="end-vector"
              size="sm"
              placeholder="0"
              required
              disabled
            />
          </div>
        </div>
        <div class="flex flex-row gap-x-2">
          <div>
            <label for="distance-x" class="mb-2 text-sm font-semibold"
              >Komponen X</label
            >
            <Input
              type="number"
              id="distance-x"
              size="sm"
              placeholder="0"
              required
              disabled
            />
          </div>
          <div>
            <label for="distance-y" class="mb-2 text-sm font-semibold"
              >Komponen Y</label
            >
            <Input
              type="number"
              id="distance-y"
              size="sm"
              placeholder="0"
              required
              disabled
            />
          </div>
        </div>
        <div>
          <label for="distance" class="mb-2  text-sm font-semibold"
            >Panjang</label
          >
          <Input
            type="number"
            id="distance"
            size="sm"
            placeholder="0"
            required
            disabled
          />
        </div>
        <div class="flex flex-wrap items-center gap-2 mt-4">
          <Button
            class="!p-2"
            on:click={() => viewVector(currentVectorIndex - 1)}
            disabled={currentVectorIndex == 0}
            ><svg
              xmlns="http://www.w3.org/2000/svg"
              fill="none"
              viewBox="0 0 24 24"
              stroke-width="1.5"
              stroke="currentColor"
              class="w-6 h-6"
            >
              <path
                stroke-linecap="round"
                stroke-linejoin="round"
                d="M10.5 19.5L3 12m0 0l7.5-7.5M3 12h18"
              />
            </svg>
          </Button>
          <Button
            class="!p-2"
            on:click={() => viewVector(currentVectorIndex + 1)}
            disabled={currentVectorIndex ==
              savedRoutes[currentRoute]?.vectors.length - 1}
            ><svg
              xmlns="http://www.w3.org/2000/svg"
              fill="none"
              viewBox="0 0 24 24"
              stroke-width="1.5"
              stroke="currentColor"
              class="w-6 h-6"
            >
              <path
                stroke-linecap="round"
                stroke-linejoin="round"
                d="M13.5 4.5L21 12m0 0l-7.5 7.5M21 12H3"
              />
            </svg>
          </Button>
        </div>
      </div>
    </div>
    <div class="w-full"><div bind:this={mapElement} class="h-screen" /></div>
  </div>
</main>

<style>
  @import "leaflet/dist/leaflet.css";
</style>
