<script setup>
import { ref } from 'vue';
import * as pdfjs from 'pdfjs-dist'
import XLSX from 'xlsx';

let file = ref(null)

let onChange = () => {
  handlePdf(file.value.files[0])
  file.value = null
}

let dragover = (event) =>  {
  event.preventDefault();
  // Add some visual fluff to show the user can drop its files
  if (!event.currentTarget.classList.contains('bg-green-300')) {
    event.currentTarget.classList.remove('bg-gray-100');
    event.currentTarget.classList.add('bg-green-300');
  }
}
let dragleave = (event) =>{
  // Clean up
  event.currentTarget.classList.add('bg-gray-100');
  event.currentTarget.classList.remove('bg-green-300');
}
let drop = (event) => {
  event.preventDefault();
  handlePdf(event.dataTransfer.files[0])
  event.currentTarget.classList.add('bg-gray-100');
  event.currentTarget.classList.remove('bg-green-300');
}

let handlePdf = (file) => {
  pdfjs.GlobalWorkerOptions.workerSrc = '//cdn.jsdelivr.net/npm/pdfjs-dist@2.9.359/build/pdf.worker.min.js';

  const reader = new FileReader();
  reader.readAsDataURL(file)

  reader.addEventListener('load', () => {
    var loadingTask = pdfjs.getDocument(reader.result);
    loadingTask.promise.then(async function (pdf) {
      function _getY(item) {
        // scaleX, scale01, scale10, scaleY, x, y
        if (item && Array.isArray(item.transform)) {
          return item.transform[4] || -1;
        }

        return -1;
      }

      const genTextContextMatrix = async (pdf, options = {}) => {
        const {onProgress, start, end} = options;

        const result = [];
        let numPage = 1;
        let numPages = 0;
        if (typeof start === 'number' && typeof end === 'number' && start < end) {
          numPage = start;
          numPages = end;
        }

        // set end
        if (typeof pdf.numPages === 'number' && numPages === 0) {
          numPages = pdf.numPages;
        }

        // page increase
        while (numPage <= numPages) {
          if (typeof onProgress === 'function') {
            onProgress({numPage, numPages});
          }

          // eslint-disable-next-line
          const page = await pdf.getPage(numPage);
          // eslint-disable-next-line
          const text = await page.getTextContent();

          if (Array.isArray(text.items)) {
            const {items} = text;
            const min = _getY(items[0]);
            let tmp = [];

            for (let i = 0; i < items.length; i += 1) {
              const y = _getY(items[i]);
              if (y <= min) {
                result.push(tmp);
                tmp = [];
              }
              tmp.push(items[i]);
            }

            if (tmp.length) result.push(tmp);
          }

          numPage += 1;
        }

        return result;
      };

      const data = await genTextContextMatrix(pdf);

      const [first = [], ...rest] = data;
      // extract text
      const header = first.map((e) => e.str);
      const ws = XLSX.utils.aoa_to_sheet(rest.map((r) => r.map((e) => e.str)), header);
      const wb = XLSX.utils.book_new();
      XLSX.utils.book_append_sheet(wb, ws, 'Sheet1');

      XLSX.writeFile(wb, file.name.replace('.pdf', '.xlsx'));
    });
  }, false);
}
</script>

<template>
  <div class="flex w-full h-screen items-center justify-center text-center" id="app">
    <div class="p-12 bg-gray-100 border border-gray-300" @dragover="dragover" @dragleave="dragleave" @drop="drop">
      <input
        type="file"
        name="fields[assetsFieldHandle][]"
        id="assetsFieldHandle"
        class="w-px h-px opacity-0 overflow-hidden absolute"
        @change="onChange"
        ref="file"
        accept=".pdf"
      />
      <label for="assetsFieldHandle" class="block cursor-pointer">
        <div>
          You can drop files in here
          or <span class="underline">click here</span> to upload your files
        </div>
      </label>
    </div>
  </div>
</template>

<style>
  @tailwind base;
  @tailwind components;
  @tailwind utilities;
</style>
