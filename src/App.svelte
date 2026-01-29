<script>
  import * as d3 from "d3";
  import RangeSlider from "svelte-range-slider-pips";
  import CirclePoint from "./lib/CirclePoint.svelte";

  import {
    getCSV,
    getGeoMultiple,
    createGeoJSONCircle,
    construct_line,
    construct_points,
  } from "./utils.js";
  import { onMount } from "svelte";
  import mapboxgl from "mapbox-gl";
  import "mapbox-gl/dist/mapbox-gl.css";

  let width = 400;
  let height = 400;
  let isOverlayVisible = true;
  let enabled = false;
  let selectedCountry = "Afghanistan";

  // remove initial div
  function removeOverlay() {
    isOverlayVisible = false;
    enabled = true;
  }

  let blah = createGeoJSONCircle([67.709953, 33.93911], 1500);
  let krava = createGeoJSONCircle([67.709953, 33.93911], 5000);

  let mapContainer;
  mapboxgl.accessToken =
    "pk.eyJ1Ijoic2FzaGFnYXJpYmFsZHkiLCJhIjoiY2xyajRlczBlMDhqMTJpcXF3dHJhdTVsNyJ9.P_6mX_qbcbxLDS1o_SxpFg";

  // load data
  let geo;
  let current_period = "2023-2024";
  let scatter;
  let geo_data;
  let map;
  let path = ["./jan26.csv", "./final_month.csv"];
  let conflict_groups;
  let cleaned_geo;
  let cleaned_geo_1;
  let sortedByDate = [];
  getCSV(path).then((data) => {
    geo = data[0];
    conflict_groups = d3.groups(geo, (d) => d.conflict_country);

    cleaned_geo_1 = geo
      .map((d) => {
        // Reject rows where any relevant field is "NA"
        if (
          d.dist_mediation_conflict === "NA" ||
          d.fatalities_best === "NA" ||
          d.latitude_con === "NA" ||
          d.longitude_con === "NA" ||
          d.iso3c === "NA"
        ) {
          return null;
        }

        const distance = +d.dist_mediation_conflict;
        const deaths = +d.fatalities_best;

        if (!isFinite(distance) || distance <= 0) {
          return null;
        }

        return {
          ...d,
          distance,
          deaths: isFinite(deaths) ? deaths : 0,
        };
      })
      .filter(Boolean);

    scatter = data[1];
    const idMap = new Map();
    let currentId = 1;

    cleaned_geo = scatter
      .map((d) => {
        if (
          d.dist_mediation_conflict === "NA" ||
          d.fatalities_best === "NA" ||
          d.latitude_con === "NA" ||
          d.longitude_con === "NA" ||
          d.iso3c === "NA"
        ) {
          return null;
        }

        const distance = +d.dist_mediation_conflict;
        const deaths = +d.fatalities_best;

        if (!isFinite(distance) || distance <= 0) {
          return null;
        }

        const year = d.YYYYMM.slice(0, 4);
        const month = d.YYYYMM.slice(4, 6);
        const formatted = `${month}-${year}`;

        // Build a composite key
        const key = `${d.conflict_country}-${distance}`;
        let id;

        if (idMap.has(key)) {
          id = idMap.get(key);
        } else {
          id = currentId++;
          idMap.set(key, id);
        }

        return {
          ...d,
          distance,
          date: formatted,
          deaths: isFinite(deaths) ? deaths : 0,
          id, // assigned ID based on shared distance & country
        };
      })
      .filter(Boolean);

    sortedByDate = cleaned_geo.sort((a, b) => {
      // Convert "MM-YYYY" to "YYYY-MM" for proper comparison
      const [aMonth, aYear] = a.date.split("-");
      const [bMonth, bYear] = b.date.split("-");

      const aDate = `${aYear}-${aMonth}`;
      const bDate = `${bYear}-${bMonth}`;

      return aDate.localeCompare(bDate);
    });
  });

  const json_path = ["geojson.json"];
  getGeoMultiple(json_path).then((geo) => {
    geo_data = geo[0];
  });

  onMount(() => {
    map = new mapboxgl.Map({
      container: mapContainer,
      style: "mapbox://styles/sashagaribaldy/cm6ktkpxn00m901s2fo0bh1e4",
      projection: "naturalEarth",
      center: [10, 10],
      zoom: 1.2,
      logoPosition: "top-right",
    });

    map.on("load", () => {
      if (conflict_groups[0][1] && geo_data) {
        const lines = construct_line(conflict_groups[0][1]);
        const points = construct_points(conflict_groups[0][1]);

        map.addSource("countries", {
          type: "geojson",
          data: geo_data,
          generateId: true, // Ensures all features have unique IDs
        });

        // Labels layer beneath polygons
        const labelLayerId = map
          .getStyle()
          .layers.find(
            (layer) => layer.type === "symbol" && layer.layout?.["text-field"],
          )?.id;

        // Add fill layer with conditional color
        map.addLayer(
          {
            id: "countries_fill",
            type: "fill",
            source: "countries",
            paint: {
              "fill-color": "steelblue",
              "fill-opacity": 0.8,
            },
            filter: ["in", "ADMIN", "Afghanistan"],
          },
          labelLayerId,
        );

        map.addSource("polygon", {
          type: "geojson",
          data: blah,
        });

        map.addLayer({
          id: "polygon",
          type: "fill",
          source: "polygon",
          paint: {
            "fill-color": "steelblue",
            "fill-opacity": 0.2,
          },
        });

        map.addSource("polygon3", {
          type: "geojson",
          data: krava,
        });

        map.addLayer({
          id: "polygon3",
          type: "fill",
          source: "polygon3",
          paint: {
            "fill-color": "steelblue",
            "fill-opacity": 0.2,
          },
        });

        map.addSource("curved-line", {
          type: "geojson",
          data: lines,
        });

        map.addLayer({
          id: "curved-line",
          type: "line",
          source: "curved-line",
          paint: {
            "line-color": "white",
            "line-width": 1,
            "line-opacity": 0.5,
          },
        });

        map.addSource("points", {
          type: "geojson",
          data: points, // output of your `construct_points()` function
        });

        map.addLayer({
          id: "point-layer",
          type: "circle",
          source: "points",
          paint: {
            "circle-radius": [
              "interpolate",
              ["linear"],
              ["get", "agreements_sum"],
              0,
              0, // agreements_sum = 0 → radius = 0
              1,
              5, // agreements_sum = 1 → radius = 5
              5,
              10, // agreements_sum = 10 → radius = 15
              10,
              20, // agreements_sum = 50 → radius = 25
            ],
            "circle-color": "#FF5722",
            "circle-opacity": 0.5,
            "circle-stroke-width": 1,
            "circle-stroke-color": "#ffffff",
          },
        });
      }
    });

    return () => map.remove();
  });

  function updateMapData(conflictData) {
    selectedCountry = conflictData;
    if (selectedCountry === "Sudan" || selectedCountry === "Yemen") {
      current_period = "2018-2024";
    } else {
      current_period = "2023-2024";
    }

    const entry = conflict_groups.find(([name]) => name === conflictData);
    let update_data = entry[1];

    if (!map || !map.isStyleLoaded()) return;

    const newLines = construct_line(update_data);
    const newPoints = construct_points(update_data);
    let newBlah = createGeoJSONCircle(
      [+update_data[0].longitude_con, +update_data[0].latitude_con],
      1500,
    );
    let newKrava = createGeoJSONCircle(
      [+update_data[0].longitude_con, +update_data[0].latitude_con],
      5000,
    );

    const lineSource = map.getSource("curved-line");
    const pointSource = map.getSource("points");
    const polygonSource = map.getSource("polygon");
    const polygon3Source = map.getSource("polygon3");

    if (lineSource && pointSource && polygonSource) {
      map.setFilter("countries_fill", ["==", "ADMIN", conflictData]);
      lineSource.setData(newLines);
      pointSource.setData(newPoints);
      polygonSource.setData(newBlah);
      polygon3Source.setData(newKrava);
    } else {
      console.warn("Map sources not found.");
    }
  }

  $: x_scale = d3
    .scaleLinear()
    .domain([0, 17000000])
    .range([80, width - 20]);

  $: y_scale = d3
    .scaleLinear()
    .domain([0, 12000])
    .range([height - 50, 10]);

  $: y_scale1 = d3
    .scaleLinear()
    .domain([0, 25000])
    .range([height - 50, 10]);

  // axes for scatterplot
  let x_axis_grp;
  let x_axis_grp1;
  let y_axis_grp;
  let y_axis_grp1;
  $: if (x_axis_grp1 && y_axis_grp1) {
    let xAxis1 = d3.axisBottom(x_scale);
    d3.select(x_axis_grp1).call(xAxis1);
    let yAxis1 = d3.axisLeft(y_scale1);
    d3.select(y_axis_grp1).call(yAxis1);
  }

  $: if (x_axis_grp && y_axis_grp) {
    let xAxis = d3.axisBottom(x_scale);
    d3.select(x_axis_grp).call(xAxis);
    let yAxis = d3.axisLeft(y_scale);
    d3.select(y_axis_grp).call(yAxis);
  }

  // // Data state
  // let dates = [];
  // let selectedDateIndex = 0;
  // let filtered_geo = [];

  // // Extract and sort dates when sortedByDate is ready
  // $: if (sortedByDate) {
  //   const dateSet = new Set(
  //     sortedByDate.map((d) => d.YYYYMM).filter((ym) => ym && ym.length > 4),
  //   );
  //   dates = Array.from(dateSet).sort();
  //   // selectedDateIndex = dates.length - 1;
  //   updateFilteredGeo();
  // }

  // function updateFilteredGeo() {
  //   const selectedDate = dates[selectedDateIndex];
  //   filtered_geo = sortedByDate.filter((d) => d.YYYYMM === selectedDate);
  //   console.log("Filtered data:", filtered_geo);
  // }

  // function formatLabel(index) {
  //   const yyyymm = dates[index];
  //   if (!yyyymm) return "";
  //   return `${yyyymm.slice(4)}-${yyyymm.slice(0, 4)}`; // MM-YYYY
  // }
</script>

<main>
  <h1>Geography in Conflict Mediation</h1>
  <div class="blog_text">
    <p>
      This interactive world map depicts the geographic distance between
      conflict mediation locations and the actual conflict zones in Afghanistan,
      Israel, Libya, and Syria from 2023-2024 and in Sudan and Yemen from
      2018-2024.
    </p>
    <p>
      The map displays arc lines that connect mediation event locations to their
      corresponding conflict zones, showing the physical distance between where
      negotiations take place and where conflicts are occurring. These include
      all events coded "M" for full mediation. Orange circles represent the
      number of agreements reached between parties, with larger circles
      indicating more agreements. These agreements are both formal peace
      agreements as found in the PA-X database and other agreements from the
      MEND database. Note that nearly 15% of agreements are the same agreement
      signed in different locations by conflict parties (shuttle diplomacy).
      Additionally, 5% of mediation events were conducted virtually, and the
      location is unknown for 20% of mediation events.
    </p>
    <p>
      In MEND, "mediation events" are defined as non-coercive facilitation of
      communication or negotiation between disputing parties to help them reach
      a mutually acceptable agreement or resolution to their conflict by an
      external third-party. Mediation always involves at least two (local)
      conflict stakeholders, at least one of them needing to be a belligerent.
    </p>
  </div>
  <div id="buttons">
    <button
      disabled={!enabled}
      class:selected={selectedCountry === "Afghanistan"}
      on:click={() => updateMapData("Afghanistan")}
    >
      Afghanistan
    </button>

    <button
      disabled={!enabled}
      class:selected={selectedCountry === "Israel"}
      on:click={() => updateMapData("Israel")}
    >
      Israel
    </button>

    <button
      disabled={!enabled}
      class:selected={selectedCountry === "Libya"}
      on:click={() => updateMapData("Libya")}
    >
      Libya
    </button>

    <button
      disabled={!enabled}
      class:selected={selectedCountry === "Sudan"}
      on:click={() => updateMapData("Sudan")}
    >
      Sudan
    </button>

    <button
      disabled={!enabled}
      class:selected={selectedCountry === "Syria"}
      on:click={() => updateMapData("Syria")}
    >
      Syria
    </button>

    <button
      disabled={!enabled}
      class:selected={selectedCountry === "Yemen"}
      on:click={() => updateMapData("Yemen")}
    >
      Yemen
    </button>
    <p>{current_period}</p>
  </div>
  <div class="map-wrapper">
    {#if isOverlayVisible}
      <div class="overlay">
        <button class="remove-overlay" on:click={removeOverlay}>
          Click to Explore
        </button>
      </div>
    {/if}
    <div id="map" bind:this={mapContainer}></div>
  </div>
  <div id="legend">
    <svg width="400px" height="75px">
      <text x="0" y="10" fill="white" font-size="10"
        >Number of Peace Agreements</text
      >
      <circle
        cx="20"
        cy="50"
        r="5"
        fill="#FF5722"
        fill-opacity="0.2"
        stroke="white"
      ></circle>
      <line
        x1="20"
        y1="45"
        x2="70"
        y2="45"
        stroke="white"
        stroke-dasharray="4,2"
      ></line>
      <text
        x="75"
        y="45"
        fill="white"
        font-size="10"
        alignment-baseline="middle">5</text
      >

      <circle
        cx="20"
        cy="45"
        r="10"
        fill="#FF5722"
        fill-opacity="0.2"
        stroke="white"
      ></circle>
      <line
        x1="20"
        y1="35"
        x2="70"
        y2="35"
        stroke="white"
        stroke-dasharray="4,2"
      ></line>
      <text
        x="75"
        y="35"
        fill="white"
        font-size="10"
        alignment-baseline="middle">10</text
      >

      <circle
        cx="20"
        cy="40"
        r="15"
        fill="#FF5722"
        fill-opacity="0.2"
        stroke="white"
      ></circle>
      <line
        x1="20"
        y1="25"
        x2="70"
        y2="25"
        stroke="white"
        stroke-dasharray="4,2"
      ></line>
      <text
        x="75"
        y="25"
        fill="white"
        font-size="10"
        alignment-baseline="middle">20</text
      >

      <text x="200" y="10" fill="white" font-size="10"
        >Distance from conflict (km)</text
      >
      <line
        x1="250"
        y1="40"
        x2="300"
        y2="40"
        stroke="white"
        stroke-dasharray="4,2"
      />
      <text x="305" y="43" fill="white" font-size="10">1,500km</text>
      <circle
        cx="250"
        cy="50"
        r="25"
        fill="steelblue"
        fill-opacity="0.2"
        stroke="none"
      />
      <line
        x1="250"
        y1="25"
        x2="300"
        y2="25"
        stroke="white"
        stroke-dasharray="4,2"
      />
      <text x="305" y="28" fill="white" font-size="10">5,000km</text>
      <circle
        cx="250"
        cy="50"
        r="10"
        fill="steelblue"
        fill-opacity="0.2"
        stroke="none"
      />
    </svg>
  </div>

  <h2 style="font-size: 16px;">
    Text: <a href="https://www.elisadamico.net/" target="_blank"
      >Elisa D'Amico</a
    >
  </h2>
  <h2 style="font-size: 16px; font-weight: normal;">
    <strong>Data</strong>: Peter, Mateja; Badanjak, Sanja; D'Amico, Elisa;
    Houghton, Kasia, 2025, "Mediation Event and Negotiators Database (MEND)",
    <a
      href="https://dataverse.harvard.edu/citation?persistentId=doi:10.7910/DVN/PYRHS6"
      target="_blank">doi.org/10.7910/DVN/PYRHS6</a
    >, Harvard Dataverse, V2.
  </h2>
  <h2 style="font-size: 16px;">
    Web & Visualizations: <a href="https://tomasvancisin.co.uk/" target="_blank"
      >Tomas Vancisin</a
    >
  </h2>
</main>

<style>
  .map-wrapper {
    position: relative;
    width: 100%;
    height: 80vh; /* or whatever height you need */
  }

  .overlay {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    width: 80%;
    height: 80vh;
    margin: 0px auto;
    background-color: rgba(255, 255, 255, 0);
    display: flex;
    border-radius: 5px;
    align-items: center;
    justify-content: center;
    z-index: 10;
  }

  .remove-overlay {
    color: black;
    width: 150px;
    height: 150px;
    border-radius: 50%;
    border: none;
    background-color: rgba(255, 255, 255, 0.8);
    font-family: "Montserrat", sans-serif;
    font-size: 20px;
    font-optical-sizing: auto;
    font-weight: 400;
    font-style: normal;
  }

  .remove-overlay:hover {
    cursor: pointer;
    background-color: red;
    color: white;
  }

  button {
    background-color: #001c23;
    border: solid 1px rgb(99, 99, 99);
    color: white;
    padding: 4px 20px;
    text-align: center;
    text-decoration: none;
    display: inline-block;
    font-size: 16px;
    border-radius: 4px;
    cursor: pointer;
  }

  button:hover {
    background-color: steelblue;
  }

  button:disabled {
    cursor: not-allowed;
  }

  main {
    display: flex;
    flex-direction: column;
    padding: 20px;
    max-width: 100%;
    box-sizing: border-box;
    color: rgb(246, 246, 234);
  }

  h1 {
    width: 80%;
    margin: 40px auto;
    margin-bottom: 0px;
    text-align: center;
  }

  h2 {
    width: 80%;
    margin: 5px auto;
    text-align: center;
    font-size: 20px;
  }

  li {
    padding: 5px;
  }

  .blog_text {
    width: 65%;
    margin: 50px auto;
    text-align: justify;
  }

  @media (max-width: 768px) {
    .blog_text,
    #buttons {
      width: 95%;
    }
  }

  #buttons {
    text-align: center;
    margin: 10px auto;
  }

  #legend {
    width: 80%;
    margin: auto;
  }

  #map,
  #chart01,
  #chart1 {
    width: 80%;
    height: 80vh;
    margin: auto;
  }

  #slider,
  .chart-toggle-buttons {
    width: 80%;
    margin: auto;
    text-align: center;
  }

  a {
    color: steelblue;
  }

  .selected {
    background-color: steelblue;
    color: white;
  }

  .chart-toggle-buttons {
    margin-bottom: 1em;
  }

  .chart-toggle-buttons button {
    background-color: #001c23;
    border: solid 1px rgb(99, 99, 99);
    color: white;
    padding: 4px 20px;
    text-align: center;
    text-decoration: none;
    display: inline-block;
    font-size: 16px;
    border-radius: 4px;
    cursor: pointer;
    transition: background-color 0.2s ease;
  }

  .chart-toggle-buttons button:hover {
    background-color: steelblue;
  }

  .chart-toggle-buttons button.active {
    background-color: steelblue;
  }

  .year-labels {
    display: flex;
    justify-content: space-between;
    width: 100%;
    color: white;
    font-size: 12px;
  }

  :global(.rsPips) {
    margin-bottom: 10px !important;
  }
</style>
