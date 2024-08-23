<script>
  import PDFViewer from "$lib/pdf/PDFViewer.svelte";
  import PDFPage from "$lib/pdf/PDFPage.svelte";

  // @ts-nocheck
  import * as pdfjs from "pdfjs-dist";
  import * as pdfWorker from "pdfjs-dist/build/pdf.worker.mjs";
  pdfjs.GlobalWorkerOptions.workerSrc =
    import.meta.url + "pdfjs-dist/build/pdf.worker.mjs";

  let pdf;
  let pages;
</script>

<svelte:head>
  <title>Home</title>
  <meta name="description" content="Svelte demo app" />
</svelte:head>

<input
  type="file"
  on:change={(e) => {
    let file = e.target.files[0];
    let fileReader = new FileReader();
    fileReader.onload = function () {
      //Step 4:turn array buffer into typed array
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

        console.log("pages", pages);
      });
    };
    //Step 3:Read the file as ArrayBuffer
    fileReader.readAsArrayBuffer(file);
  }}
/>

<!-- {#if typedarray}
  <PDFViewer src={typedarray} />
{/if} -->

{#if pages}
  <PDFPage page={pages[0]}><slot />{" " + 1 + "/" + pages.length}</PDFPage>
{/if}

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
