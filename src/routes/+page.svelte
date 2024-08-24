<script lang="ts">
  import PDFPage from "$lib/pdf/PDFPage.svelte";

  // @ts-nocheck
  import * as pdfjs from "pdfjs-dist";
  import * as pdfWorker from "pdfjs-dist/build/pdf.worker.mjs";
  pdfjs.GlobalWorkerOptions.workerSrc =
    import.meta.url + "pdfjs-dist/build/pdf.worker.mjs";

  let pdf;
  let pages;
  let pageNumber = 1;
  let fitToPage = true;
  let page;

  let reference;

  let radius = 100;

  //let selected = undefined;
  let selected = undefined;
  let analysis = [];

  let doRender = () => {
    console.log("Callback not yet installed");
  };

  //https://en.wikipedia.org/wiki/Haversine_formula
  const hav = (theta) => {
    return (1 - Math.cos((theta * Math.PI) / 180)) / 2;
  };

  const hav2 = (a, b) => {
    const deltaLat = b[0] - a[0];
    const deltaLong = b[1] - a[1];
    return (
      hav(deltaLat) +
      Math.cos((a[0] * Math.PI) / 180) *
        Math.cos((b[0] * Math.PI) / 180) *
        hav(deltaLong)
    );
  };

  const r = 6371000; //earth radius
  const dist = (a, b) => {
    return 2 * r * Math.asin(Math.sqrt(hav2(a, b)));
  };

  // https://codeplea.com/triangular-interpolation
  type Point = [number, number];
  const baycentricWeights: (p: Point, t: Point[]) => number[] = (p, t) => {
    const w1 =
      ((t[1][1] - t[2][1]) * (p[0] - t[2][0]) +
        (t[2][0] - t[1][0]) * (p[1] - t[2][1])) /
      ((t[1][1] - t[2][1]) * (t[0][0] - t[2][0]) +
        (t[2][0] - t[1][0]) * (t[0][1] - t[2][1]));

    const w2 =
      ((t[2][1] - t[0][1]) * (p[0] - t[2][0]) +
        (t[0][0] - t[2][0]) * (p[1] - t[2][1])) /
      ((t[1][1] - t[2][1]) * (t[0][0] - t[2][0]) +
        (t[2][0] - t[1][0]) * (t[0][1] - t[2][1]));

    const w3 = 1 - w1 - w2;

    return [w1, w2, w3];
  };
  type Vector = number[];
  export const scaleV: (s: number, v: Vector) => Vector = (s, v) => {
    let result: Vector = [];
    for (let i = 0; i < v.length; i++) {
      result[i] = s * v[i];
    }
    return result;
  };
  export const addV: (a: Vector, b: Vector) => Vector = (a, b) => {
    if (a.length != b.length) {
      throw new Error("Vectors not the same length");
    }

    let result: Vector = [];
    for (let i = 0; i < a.length; i++) {
      result[i] = a[i] + b[i];
    }
    return result;
  };

  const interpolateFromTriangle: (
    p: Point,
    trianglePoints: Point[],
    triangleVectors: Vector[]
  ) => Vector = (p, trianglePoints, triangleVectors) => {
    const weights = baycentricWeights(p, trianglePoints);

    const interpolation = addV(
      addV(
        scaleV(weights[0], triangleVectors[0]),
        scaleV(weights[1], triangleVectors[1])
      ),
      scaleV(weights[2], triangleVectors[2])
    );

    return interpolation;
  };

  const centerOfRect = (rect) => {
    return [(rect[0] + rect[2]) / 2, (rect[1] + rect[3]) / 2];
  };

  $: {
    if (pages && pages[pageNumber - 1]) {
      pages[pageNumber - 1].getAnnotations().then((annotations) => {
        console.log("page annotations", annotations);

        const referencePoints = annotations
          .filter((i) => i.contentsObj.str.startsWith("#techpdf"))
          .map((i) => [i.rect[0], i.rect[1]]);
        //console.log("reference points", referencePoints);

        const referenceVectors = annotations
          .filter((i) => i.contentsObj.str.startsWith("#techpdf"))
          .map((i) =>
            i.contentsObj.str
              .replace("#techpdf", "")
              .split(",")
              .map((j) => Number(j))
          );
        //console.log("reference vectors", referenceVectors);

        reference = {
          points: referencePoints,
          vectors: referenceVectors,
        };

        analysis = annotations
          .filter((i) => !i.contentsObj.str.startsWith("#techpdf"))
          .map((i) => {
            return {
              str: i.contentsObj.str,
              pos: interpolateFromTriangle(
                centerOfRect(i.rect),
                referencePoints,
                referenceVectors
              ),
              rect: i.rect,
            };
          });

        // defer rendering untill we have annotations
        page = pages[pageNumber - 1];
      });
    }
  }

  $: {
    console.log("analysis", analysis);
  }
</script>

<svelte:head>
  <title>techpdf</title>
  <meta name="description" content="techpdf" />
</svelte:head>

<div style="margin-bottom: 2em;">
  <a href="https://github.com/jimvella/techpdf"
    >https://github.com/jimvella/techpdf</a
  >
</div>

<input
  type="file"
  on:change={(e) => {
    let file = e.target.files[0];
    let fileReader = new FileReader();
    fileReader.onload = function () {
      let typedarray = new Uint8Array(this.result);

      const loadingTask = pdfjs.getDocument(typedarray);
      loadingTask.promise.then(function (i) {
        pdf = i;
        console.log("pdf", pdf);

        pdf.getData().then((i) => {
          console.log("data", i);
        });

        Promise.all(
          Array(pdf._pdfInfo.numPages)
            .fill()
            .map((item, index) => {
              const pageNumber = index + 1;
              return pdf.getPage(pageNumber);
            })
        ).then((i) => (pages = i));
      });
    };
    //Step 3:Read the file as ArrayBuffer
    fileReader.readAsArrayBuffer(file);
  }}
/>
page <input type="number" bind:value={pageNumber} />
<button disabled={!fitToPage} on:click={() => (fitToPage = false)}
  >Full Size</button
><button disabled={fitToPage} on:click={() => (fitToPage = true)}
  >Fit to page</button
>

{#if page}
  <div>
    <PDFPage
      {page}
      style={fitToPage ? "width: 100%" : ""}
      afterRender={(context, canvas) => {
        if (selected != undefined) {
          console.log("selected", analysis[selected]);
          console.log("page", page);

          const viewport = page.getViewport();
          const pdfWidth = viewport.viewBox[2];
          const pdfHeight = viewport.viewBox[3];
          const canvasWidth = canvas.width;
          const canvasHeight = canvas.height;
          const isOdd = (n) => {
            return n % 2;
          };
          const pdfToCanvasCoords = (points) => {
            // Canvas origin is top left,
            // pdf origin is bottom left
            let result = [];
            for (let i = 0; i < points.length; i++) {
              if (isOdd(i)) {
                // y
                result.push(
                  ((pdfHeight - points[i]) * canvasHeight) / pdfHeight
                );
              } else {
                // x cord
                result.push((points[i] * canvasWidth) / pdfWidth);
              }
            }
            return result;
          };

          let x = pdfToCanvasCoords(analysis[selected].rect);

          // m to canvas units
          // canvas distance between reference points
          let a = pdfToCanvasCoords(reference.points[0]);
          let b = pdfToCanvasCoords(reference.points[1]);
          const canvasDistance = Math.sqrt(
            Math.pow(b[0] - a[0], 2) + Math.pow(b[1] - a[1], 2)
          );

          // meters
          const actualDistance = dist(
            reference.vectors[0],
            reference.vectors[1]
          );

          //context.fillStyle = "rgb(0,0,255,0.2)";
          //context.fillRect(x[0], x[1], x[2] - x[0], x[3] - x[1]);

          context.beginPath();
          context.arc(
            (x[0] + x[2]) / 2,
            (x[1] + x[3]) / 2,
            (radius / actualDistance) * canvasDistance,
            0,
            2 * Math.PI
          );
          context.fillStyle = "rgb(0,0,255,0.2)";
          context.fill();
          //context.lineWidth = 4;
          //context.strokeStyle = "blue";
          //context.stroke();
        }
      }}
      doRenderCallback={(f) => (doRender = f)}
    />
  </div>
{/if}

<div>
  Radius (m) <input
    type="number"
    bind:value={radius}
    on:change={() => {
      doRender();
    }}
  />
</div>
<table>
  <tr><th>Description</th><th>Position</th></tr>
  {#each analysis as i, index}
    <tr
      on:click={() => {
        selected = index;
        console.log("calling");
        doRender();
      }}
      style={selected != undefined && selected == index
        ? "background-color: orange"
        : ""}
    >
      <td>{i.str}</td>
      <td>{i.pos.join(", ")}</td>
    </tr>
  {/each}
</table>

<style>
  section {
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    flex: 0.6;
  }

  h1 {
    width: 100%;
  }

  .welcome {
    display: block;
    position: relative;
    width: 100%;
    height: 0;
    padding: 0 0 calc(100% * 495 / 2048) 0;
  }

  .welcome img {
    position: absolute;
    width: 100%;
    height: 100%;
    top: 0;
    display: block;
  }
</style>
