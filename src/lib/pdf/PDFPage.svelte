<script lang="ts">
  import * as pdfjs from "pdfjs-dist";

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

  const interpolateTriangle3: (
    p: Point,
    triangle: Point[],
    vectors: Vector[]
  ) => Vector = (p, triangle, vectors) => {
    const weights = baycentricWeights(p, triangle);

    const interpolation = addV(
      addV(scaleV(weights[0], vectors[0]), scaleV(weights[1], vectors[1])),
      scaleV(weights[2], vectors[2])
    );

    return interpolation;
  };

  //export let promise: Promise<pdfjs.PDFPageProxy>;
  export let page: pdfjs.PDFPageProxy;
  export let style = "";

  let canvas;

  let annotations;

  // promise.then((item) => {
  //   console.log("page", item);

  //   item.getAnnotations().then((i) => {
  //     annotations = i;
  //     page = item;

  //     console.log("annotations", annotations);
  //   });
  // });

  $: {
    if (canvas) {
      const context = canvas.getContext("2d");

      const scale = 1.5;
      const viewport = page.getViewport({ scale: scale });
      // Support HiDPI-screens.
      const outputScale = window.devicePixelRatio || 1;

      canvas.width = Math.floor(viewport.width * outputScale);
      canvas.height = Math.floor(viewport.height * outputScale);
      //canvas.style.width = Math.floor(viewport.width) + "px";
      //canvas.style.height = Math.floor(viewport.height) + "px";

      const transform =
        outputScale !== 1 ? [outputScale, 0, 0, outputScale, 0, 0] : null;

      const renderContext = {
        canvasContext: context,
        transform: transform,
        viewport: viewport,
      };
      const r = page.render(renderContext);
      r.promise.then(() => {
        console.log("viewport", page.getViewport());
        console.log("canvas", {
          width: canvas.width,
          height: canvas.height,
        });

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
              result.push(((pdfHeight - points[i]) * canvasHeight) / pdfHeight);
            } else {
              // x cord
              result.push((points[i] * canvasWidth) / pdfWidth);
            }
          }
          return result;
        };

        const referencePoints = annotations.filter((i) => {
          return i.contentsObj.str.startsWith("-27");
        });

        context.beginPath();
        {
          const p = pdfToCanvasCoords(referencePoints[0].rect.slice(0, 2));
          context.moveTo(p[0], p[1]);
        }
        {
          const p = pdfToCanvasCoords(referencePoints[1].rect.slice(0, 2));
          context.lineTo(p[0], p[1]);
        }
        {
          const p = pdfToCanvasCoords(referencePoints[2].rect.slice(0, 2));
          context.lineTo(p[0], p[1]);
        }
        {
          const p = pdfToCanvasCoords(referencePoints[0].rect.slice(0, 2));
          context.lineTo(p[0], p[1]);
        }
        context.strokeStyle = "blue";
        context.stroke();

        const targetPoints = annotations.filter((i) => {
          return i.contentsObj.str.startsWith("R");
        });
        console.log("target", targetPoints);

        const v = referencePoints.map((i) =>
          i.contentsObj.str.split(",").map((j) => Number(j))
        );
        console.log("v", v);

        const results = targetPoints.map((i) => {
          return interpolateTriangle3(
            i.rect.slice(0, 2),
            referencePoints.map((j) => j.rect.slice(0, 2)),
            v
          );
        });

        console.log("results", results);
        console.log("dist", dist(results[0], results[1]));
      });
    }
  }
</script>

<!-- <div class="pageContainer" style="text-align: center;">
  <div>
    <div style="position: absolute;">
      <slot />
    </div>
    {#if page} -->
<!-- <canvas bind:this={canvas} style="width: 100%; height: auto;" /> -->
<canvas bind:this={canvas} {style} />

<!-- {:else}
      Loading...
    {/if}
  </div>
</div> -->

<style>
  .pageContainer {
    width: 45%;
  }

  @media screen and (max-width: 1240px) {
    .pageContainer {
      width: 100%;
    }
  }

  @media print and (max-width: 800px) {
    .pageContainer {
      width: 100%;
    }
  }
</style>
