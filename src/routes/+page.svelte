<script lang="ts">
  import PDFPage from "$lib/pdf/PDFPage.svelte";
  import { AnnotationFactory, AnnotationIcon } from "annotpdf";

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
  let fileName = "";

  let reference;

  let radius = 100;

  //let selected = undefined;
  let selected = undefined;
  let selectedRectText = "";
  let selectedProjectedDimentions = "";

  let analysis = [];

  let categories = [];
  let enabledCategories = {};

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

  const dimentions = (rect) => {
    const xy = interpolateFromTriangle(
      [rect[0], rect[1]],
      reference.points,
      reference.vectors
    );

    const x1y1 = interpolateFromTriangle(
      [rect[2], rect[3]],
      reference.points,
      reference.vectors
    );

    const x1y = interpolateFromTriangle(
      [rect[2], rect[1]],
      reference.points,
      reference.vectors
    );

    return [dist(xy, x1y), dist(x1y, x1y1)];
  };

  // $: {
  //   if (pages && pages[pageNumber - 1]) {
  //   }
  // }

  const load: (typedarray: Uint8Array) => Promise<any> = (typedarray) => {
    const loadingTask = pdfjs.getDocument(typedarray);
    return new Promise((resolve, reject) => {
      loadingTask.promise.then(function (i) {
        pdf = i;

        Promise.all(
          Array(pdf._pdfInfo.numPages)
            .fill()
            .map((item, index) => {
              const pageNumber = index + 1;
              return pdf.getPage(pageNumber);
            })
        ).then((i) => {
          pages = i;

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

            reference = {
              points: referencePoints,
              vectors: referenceVectors,
            };

            // For editing
            // https://github.com/highkite/pdfAnnotate

            analysis = annotations
              .filter((i) => !i.contentsObj.str.startsWith("#techpdf"))
              .filter((i) => i.subtype.toLowerCase() != "Popup".toLowerCase())
              .map((i) => {
                return {
                  str: i.contentsObj.str,
                  pos: interpolateFromTriangle(
                    centerOfRect(i.rect),
                    referencePoints,
                    referenceVectors
                  ),
                  rect: i.rect,
                  subtype: i.subtype,
                  annotation: i,
                };
              });

            categories = [
              ...new Set(
                annotations
                  .filter((i) => !i.contentsObj.str.startsWith("#techpdf"))
                  .map((i) => i.contentsObj.str)
                  .filter((i) => i.includes("#"))
                  .flatMap((i) => {
                    const r = i.split(/\s+/);
                    return r;
                  })
                  .filter((i) => i.startsWith("#"))
                  .map((i) => i.replace("#", ""))
              ),
            ];

            // defer rendering untill we have annotations
            page = pages[pageNumber - 1];

            resolve(null);
          });
        });
      });
    });
  };

  const downloadURL = (data, fileName) => {
    const a = document.createElement("a");
    a.href = data;
    a.download = fileName;
    document.body.appendChild(a);
    a.style.display = "none";
    a.click();
    a.remove();
  };

  const downloadBlob = (data, fileName, mimeType) => {
    const blob = new Blob([data], {
      type: mimeType,
    });
    const url = window.URL.createObjectURL(blob);
    downloadURL(url, fileName);
    setTimeout(() => window.URL.revokeObjectURL(url), 1000);
  };

  $: {
    console.log("categories", categories);
    categories.forEach((i) => {
      if (enabledCategories[i] == undefined) {
        enabledCategories[i] = true;
      }
    });
  }

  $: {
    console.log("analysis", analysis);
  }

  $: {
    if (selected) {
      const rect = selectedRectText.split(",").map((i) => Number(i));
      selectedProjectedDimentions = dimentions(rect)
        .map((i) => +i.toPrecision(3) + "m")
        .join(" x ");
    }
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
<!-- <div>
  <button
    on:click={() => {
      console.log("Test");
      pdf.getData().then((data) => {
        let pdfFactory = new AnnotationFactory(data);
        pdfFactory.createCircleAnnotation({
          page: 0,
          rect: [
            88.70719 + 100,
            511.8047 + 100,
            120.1761 + 100,
            526.1814 + 100,
          ],
          contents: "Test circle",
          author: "Max",
          color: { r: 0, g: 255, b: 0 },
          //opacity: 0.5,
        });
        load(pdfFactory.write()).then(() => {
          console.log("loaded");
          doRender();
        });
      });
    }}>Test</button
  >
</div> -->
<input
  type="file"
  on:change={(e) => {
    let file = e.target.files[0];
    fileName = file.name;
    let fileReader = new FileReader();
    fileReader.onload = function () {
      load(new Uint8Array(this.result));
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
<button
  on:click={() => {
    pdf.getData().then((data) => {
      downloadBlob(data, fileName, "application/pdf");
    });
  }}>Download</button
>

{#if page}
  <div>
    <PDFPage
      {page}
      style={fitToPage ? "width: 100%" : ""}
      afterRender={(context, canvas) => {
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

        // m to canvas units
        // canvas distance between reference points
        let a = pdfToCanvasCoords(reference.points[0]);
        let b = pdfToCanvasCoords(reference.points[1]);
        const canvasDistance = Math.sqrt(
          Math.pow(b[0] - a[0], 2) + Math.pow(b[1] - a[1], 2)
        );

        const pdfDistance = Math.sqrt(
          Math.pow(reference.points[1][0] - reference.points[0][0], 2) +
            Math.pow(reference.points[1][1] - reference.points[0][1], 2)
        );

        // meters
        const actualDistance = dist(reference.vectors[0], reference.vectors[1]);

        if (selected != undefined) {
          console.log("selected", analysis[selected]);
          console.log("page", page);

          let x = pdfToCanvasCoords(analysis[selected].rect);

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

        // Draw categories
        categories.forEach((category) => {
          if (enabledCategories[category]) {
            console.log("drawing " + category);
            analysis
              .filter((i) => i.str.includes("#" + category + " "))
              .forEach((element) => {
                const spec = element.str
                  .substring(element.str.indexOf("#" + category))
                  .split(/\s+/);

                if (element.subtype == "Circle") {
                  const w = element.rect[2] - element.rect[0];
                  const h = element.rect[1] - element.rect[3];
                  const r = Math.max(w, h) / 2;
                  const rm = (r / pdfDistance) * actualDistance;

                  let x = pdfToCanvasCoords(element.rect);

                  // context.lineWidth =
                  //   Number(spec[1]) / actualDistance + canvasDistance;

                  context.beginPath();
                  context.arc(
                    (x[0] + x[2]) / 2,
                    (x[1] + x[3]) / 2,
                    ((rm + Number(spec[1])) / actualDistance) * canvasDistance,
                    0,
                    2 * Math.PI
                  );
                  context.fillStyle = spec[2];

                  context.fill();
                } else {
                  let x = pdfToCanvasCoords(element.rect);

                  const margin =
                    (Number(spec[1]) / actualDistance) * canvasDistance;

                  const w = x[2] - x[0] + margin + margin;
                  const h = x[3] - x[1] - margin - margin;

                  context.fillStyle = spec[2];
                  console.log("fillStyle", spec[2]);
                  context.fillRect(x[0] - margin, x[1] + margin, w, h);
                }
              });
          }
        });

        // Draw selection edit
        if (selected) {
          console.log("Draw selected edit");
          const rect = selectedRectText.split(",").map((i) => Number(i));

          let x = pdfToCanvasCoords(rect);
          context.strokeStyle = "rgb(255,0,255,0.5)";
          context.lineWidth = 10;
          context.strokeRect(x[0], x[1], x[2] - x[0], x[3] - x[1]);
        }
      }}
      doRenderCallback={(f) => (doRender = f)}
    />
  </div>
{/if}

<div style="margin-bottom: 1em;">
  {#each categories as i, index}
    <div>
      {i}
      <input
        type="checkbox"
        bind:checked={enabledCategories[i]}
        on:change={() => {
          doRender();
        }}
      />
    </div>
  {/each}
</div>

<div>
  Radius (m) <input
    type="number"
    bind:value={radius}
    on:change={() => {
      doRender();
    }}
  />
</div>

<div style="display: flex;">
  <div>
    <table>
      <tr><th>Description</th><th>Position</th></tr>
      {#each analysis as i, index}
        <tr
          on:click={() => {
            if (selected == index) {
              selected = undefined;
            } else {
              selected = index;
              selectedRectText = analysis[selected].rect.join(", ");
            }
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
  </div>
  <div style="width: 100%; background-color:lightgray">
    {#if selected}
      {analysis[selected].rect}
      <br />
      {dimentions(analysis[selected].rect)
        .map((i) => +i.toPrecision(3) + "m")
        .join(" x ")}

      <div>
        rect <input
          type="text"
          style="width: 100%; box-sizing: border-box"
          bind:value={selectedRectText}
          on:input={() => {
            doRender();
          }}
        />
      </div>
      <div>
        Projected: {selectedProjectedDimentions}
      </div>
      <div>
        <button
          on:click={() => {
            pdf.getData().then((data) => {
              let pdfFactory = new AnnotationFactory(data);

              if (analysis[selected].annotation.subtype == "Square") {
                pdfFactory.createSquareAnnotation({
                  page: pageNumber - 1,
                  rect: selectedRectText.split(",").map((i) => Number(i)),
                  contents: analysis[selected].annotation.contentsObj.str,
                  author: "techpdf",
                  color: {
                    r: analysis[selected].annotation.color[0],
                    g: analysis[selected].annotation.color[1],
                    b: analysis[selected].annotation.color[2],
                  },
                  //opacity: 0.5,
                });
              } else {
                pdfFactory.createCircleAnnotation({
                  page: pageNumber - 1,
                  rect: selectedRectText.split(",").map((i) => Number(i)),
                  contents: analysis[selected].annotation.contentsObj.str,
                  author: "techpdf",
                  color: {
                    r: analysis[selected].annotation.color[0],
                    g: analysis[selected].annotation.color[1],
                    b: analysis[selected].annotation.color[2],
                  },
                });
              }

              load(pdfFactory.write()).then(() => {
                console.log("loaded");
                doRender();
              });

              // For the deletion ids to line up - need to refactor in terms of the pdfAnnotate library
              // https://github.com/highkite/pdfAnnotate
              // pdfFactory.getAnnotations().then((i) => {
              //   console.log("factory annotations", i);
              // });

              // pdfFactory
              //   .deleteAnnotation(analysis[selected].annotation.id)
              //   .then(() => {
              //     console.log("deleted");

              //   });
            });
          }}>Update</button
        >
      </div>
    {/if}
  </div>
</div>

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
