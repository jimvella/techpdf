<script lang="ts">
  // @ts-nocheck
  import * as pdfjs from "pdfjs-dist";
  import * as pdfWorker from "pdfjs-dist/build/pdf.worker.mjs";
  import PdfPage from "./PDFPage.svelte";
  pdfjs.GlobalWorkerOptions.workerSrc =
    import.meta.url + "pdfjs-dist/build/pdf.worker.mjs";

  export let src;

  let pdf;
  let pages: Promise<pdfjs.PDFPageProxy>[];

  if (src) {
    const loadingTask = pdfjs.getDocument(src);
    loadingTask.promise.then(function (i) {
      pdf = i;
      console.log("pdf", pdf);

      pdf.getData().then((i) => {
        console.log("data", i);
      });

      pages = Array(pdf._pdfInfo.numPages)
        .fill()
        .map((item, index) => {
          const pageNumber = index + 1;
          return pdf.getPage(pageNumber);
        });
      console.log("pages", pages);

      // pages.forEach((page, i) => {
      //   const pageNumber = i + 1;
      //   pdf.getPage(pageNumber).then((j) => {
      //     console.log("page loaded", j);
      //     pages[i] = j;
      //   });
      // });
    });
  }
</script>

{#if pages}
  {#each pages as page, i}
    {#if page}
      <PdfPage promise={page}
        ><slot />{" " + (i + 1) + "/" + pages.length}</PdfPage
      >
    {:else}
      <div>loading {i}</div>
    {/if}
  {/each}
{:else}
  <div>Loading...</div>
{/if}
