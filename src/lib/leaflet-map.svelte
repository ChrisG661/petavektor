<script>
  import { onMount, onDestroy } from "svelte";
  import { browser } from "$app/environment";

  let mapElement;
  let map;

  onMount(async () => {
    if (browser) {
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
      const L = await import("leaflet");

      map = L.map(mapElement).setView([1.438654, 124.790774], 13);

      L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {
        attribution:
          '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
      }).addTo(map);

      L.marker([1.437757, 124.790186]).addTo(map).bindPopup("Rumah");
      //.openPopup();

      async function getRouteGeometry(start, destination) {
        const geometry = await fetch(
          `https://router.project-osrm.org/route/v1/driving/${start[1]},${start[0]};${destination[1]},${destination[0]}?generate_hints=false&alternatives=false&geometries=geojson&overview=full`,
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
            console.log(coordinates);
            return coordinates;
          })
          .catch((error) => {
            console.error(error);
          });
        return geometry;
      }
      async function drawRoute(start, destination) {
        await getRouteGeometry(start, destination).then((coordinates) => {
          L.polyline(coordinates, { color: "" }).addTo(map);
        });
      }

      function drawRoutePoint(coordinate) {
        L.circle(coordinate, { icon: L.divIcon() }).addTo(map);
      }
      drawRoute([1.437533, 124.79023], [1.46146, 124.837412]);
      //drawRoutePoint([1.46146, 124.837412]);
    }
  });

  onDestroy(async () => {
    if (map) {
      console.log("Unloading Leaflet map.");
      map.remove();
    }
  });
</script>

<div bind:this={mapElement} class="h-screen" />

<style>
  @import "leaflet/dist/leaflet.css";
</style>
