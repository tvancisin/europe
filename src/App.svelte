<script>
  import { onMount } from "svelte";
  import * as d3 from "d3";
  import Hls from "hls.js";

  let audioElement;

  // Stream URLs for each country
  const streams = {
    uk: "https://as-hls-ww-live.akamaized.net/pool_904/live/ww/bbc_6music/bbc_6music.isml/bbc_6music-audio%3d96000.norewind.m3u8",
    austria: "https://orf-live.ors-shoutcast.at/fm4-q2a",
    portugal: "https://radiocast.rtp.pt/antena380a.mp3",
    slovakia: "https://icecast.stv.livebox.sk/fm_128.mp3",
    france: "https://icecast.radiofrance.fr/fip-midfi.mp3?id=radiofrance",
    spain: "https://rtvelivestream.akamaized.net/rtvesec/rne/rne_r3_main.m3u8",
    ireland: "https://icecast1.rte.ie/2xm",
    iceland: "http://netradio.ruv.is/ras2.aac",
    netherlands: "https://icecast.omroep.nl/3fm-bb-aac",
    czech: "https://rozhlas.stream/radio_wave_low.aac",
    switzerland: "https://stream.srg-ssr.ch/m/drsvirus/aacp_96",
    italy: "https://radiodueindie-live.akamaized.net/hls/live/2032593/radiodueindie/radiodueindie/radio2indie_256/chunklist.m3u8",
    germany: "https://st03.sslstream.dlf.de/dlf/03/high/aac/stream.aac",
    denmark: "https://drliveradio.akamaized.net/hls/live/2022411/p6beat/masterab.m3u8",
    poland: "http://mp3.polskieradio.pl:8904/;",
    hungary: "https://icast.connectmedia.hu/4738/mr2.mp3",
    slovenia: "https://mp3.rtvslo.si/val202",
    norway: "https://lyd.nrk.no/nrk_radio_p13_aac_h",
    sweden: "https://http-live.sr.se/p3-aac-192",
    finland: "https://icecast.live.yle.fi/radio/YleX/icecast.audio",
    estonia: "https://sb.err.ee/live/raadio2.m3u8"
  };

  let width, height;
  let transform = d3.zoomIdentity;

  let geojson;
  d3.json("europe.json").then((data) => (geojson = data));

  $: projection = d3.geoOrthographic().fitSize([width, height], geojson);
  $: pathGenerator = d3.geoPath(projection);

  let countries = [];
  $: if (geojson)
    countries = geojson.features.map((feature) => {
      return {
        ...feature,
        path: pathGenerator(feature),
        bounds: pathGenerator.bounds(feature),
        centroid: pathGenerator.centroid(feature), // Get the country's central point
      };
    });

  function smoothZoom(newTransform) {
    const interpolate = d3.interpolateTransformSvg(transform, newTransform);
    d3.transition()
      .duration(750)
      .tween("zoom", () => (t) => {
        transform = interpolate(t);
      });
  }

  let selectedCountry;
  function handleClick(e) {
    selectedCountry = e.properties.NAME;
    if (selectedCountry == "United Kingdom") {
      selectedCountry = "uk";
    }
    if (selectedCountry == "Czech Republic") {
      selectedCountry = "czech"
    }

    const streamUrl = streams[selectedCountry.toLowerCase()];
    if (streamUrl) {
      playStream(streamUrl);
    } else {
      console.error(`No stream available for ${selectedCountry}`);
    }

    const [[x0, y0], [x1, y1]] = e.bounds;
    const newTransform = d3.zoomIdentity
      .translate(width / 2, height / 2)
      .scale(Math.min(8, 0.9 / Math.max((x1 - x0) / width, (y1 - y0) / height)))
      .translate(-(x0 + x1) / 2, -(y0 + y1) / 2);

    smoothZoom(newTransform);
  }

  function playStream(url) {
    // Check if the stream is HLS or MP3
    if (url.endsWith(".m3u8")) {
      // Handle HLS streams
      if (Hls.isSupported()) {
        const hls = new Hls();
        hls.loadSource(url);
        hls.attachMedia(audioElement);
        hls.on(Hls.Events.MANIFEST_PARSED, () => {
          console.log("HLS Manifest loaded, ready to play.");
          audioElement.play();
        });
        hls.on(Hls.Events.ERROR, (event, data) => {
          console.error("HLS Error:", data);
        });
      } else if (audioElement.canPlayType("application/vnd.apple.mpegurl")) {
        // Safari handles HLS natively
        audioElement.src = url;
        audioElement.play();
      } else {
        console.error("Your browser does not support HLS playback.");
      }
    } else {
      // Handle MP3 streams
      audioElement.src = url;
      audioElement.play();
    }
  }

  function handleEscape(event) {
    if (event.key === "Escape") {
      smoothZoom(d3.zoomIdentity);
    }
  }

  onMount(() => {
    window.addEventListener("keydown", handleEscape);
    return () => window.removeEventListener("keydown", handleEscape);
  });

  function handleMouseEnter(event) {
    event.target.style.fill = "red";
  }

  function handleMouseLeave(event) {
    event.target.style.fill = "#1B1A1B";
  }
</script>

{#if countries}
  <main bind:clientWidth={width} bind:clientHeight={height}>
    <audio bind:this={audioElement} controls autoplay></audio>

    <svg {width} {height}>
      <g {transform}>
        {#each countries as country}
          <path
            fill="#1B1A1B"
            stroke="#333333"
            stroke-width="0.5"
            d={country.path}
            on:click={() => handleClick(country)}
            on:mouseenter={handleMouseEnter}
            on:mouseleave={handleMouseLeave}
          ></path>
          <image
            on:click={() => handleClick(country)}
            href={country.properties.radio}
            x={country.centroid[0] - 10}
            y={country.centroid[1] - 10}
            height="25"
          ></image>
        {/each}
      </g>
    </svg>
  </main>
{/if}

<style>
  main {
    width: 100vw;
    height: 100vh;
    cursor: pointer;
    overflow: hidden;
  }
  audio {
    position: absolute;
    right: 5px;
    top: 5px;
  }
</style>
