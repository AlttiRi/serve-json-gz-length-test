<template>
  <label>
    <input type="checkbox" v-model="GZipped"> GZipped
  </label>

  <hr>
  <span class="receivedLength">{{bytesToSizeWinLike(receivedLength)}}</span>
  of
  <span class="contentLength">{{bytesToSizeWinLike(contentLength)}}</span>
  <hr>
  <span class="percent">{{percent}} %</span> downloaded.
  <hr>
  <div class="warning">{{warning}}</div>
</template>

<script setup>
import {ref, watchEffect} from "vue";


const GZipped = ref(true);
const receivedLength = ref(0);
const contentLength = ref(0);
const percent = ref("0");

const warning = ref("");


watchEffect(async () => {
  receivedLength.value = 0;
  contentLength.value = 0;
  percent.value = "0";

  const response = await fetch("./[.dir-scan] 2022.01.09.json" + (GZipped.value ? ".gz" : ""));
  loadingProxyResponse({response, onProgress: (r, t) => {
      onLoadingProgressHTML(r, t);
      onLoadingProgressConsoleLog(r, t);
    }});
});

watchEffect(() => {
  if (parseFloat(percent.value) > 100) {
    warning.value = "% is more than 100";
  } else  {
    warning.value = "";
  }
});


function onLoadingProgressHTML(_receivedLength, _contentLength) {
  receivedLength.value = _receivedLength;
  contentLength.value = _contentLength;
  percent.value = (_receivedLength / _contentLength * 100 + "").substring(0, 5);
}


function bytesToSizeWinLike(bytes) {
  if (bytes < 1024) { return bytes + " B"; }
  const sizes = ["B", "KB", "MB", "GB", "TB", "PB", "EB", "ZB", "YB"];
  let i = Math.floor(Math.log(bytes) / Math.log(1024));
  let result = bytes / Math.pow(1024, i);
  if (result >= 1000) {
    i++;
    result /= 1024;
  }
  return toTruncPrecision3(result) + " " + sizes[i];
}
function toTruncPrecision3(number) {
  let result;
  if (number < 10) {
    result = Math.trunc(number * 100) / 100;
  } else if (number < 100) {
    result = Math.trunc(number * 10) / 10;
  } else if (number < 1000) {
    result = Math.trunc(number);
  }
  if (number < 0.1) {
    return result.toPrecision(1);
  } else if (number < 1) {
    return result.toPrecision(2);
  }
  return result.toPrecision(3);
}


function loadingProxyResponse({response, onProgress, contentLength, contentType}) {
  if (!onProgress) {
    return response;
  }
  contentLength = contentLength || parseInt(response.headers.get("Content-Length"));
  contentType = contentType || response.headers.get("Content-Type");

  console.log("Content-Length: ", contentLength);
  if (isNaN(contentLength)) {
    console.warn("Content-Length is NaN:", contentLength);
    return response;
  }

  let receivedLength = 0;
  const reader = response.body.getReader();
  const readableStream = new ReadableStream({
    async start(controller) {
      while (true) {
        const {done, /** @type {Uint8Array} */ value} = await reader.read();
        if (done) {
          break;
        }
        receivedLength += value.length;
        try {
          onProgress(receivedLength, contentLength, value, contentType);
        } catch (e) {
          console.error("onProgress error:", e);
        }
        controller.enqueue(value);
      }
      controller.close();
      reader.releaseLock();
    },
    cancel() {
      reader.cancel();
    }
  });

  // https://developer.mozilla.org/en-US/docs/Web/API/Response/Response
  // console.log(response);
  return new Response(readableStream, {
    status: response.status,
    statusText: response.statusText,
    headers: response.headers,
  });
}

const logDebouncer = getDebounce();
function getDebounce() {
  let cb;
  let timer;
  return function debounce(callback, reset = false) {
    if (!cb || reset) {
      callback();
      cb = true;
      clearTimeout(timer);
      timer = null;
      return;
    }
    cb = callback;
    if (timer) {
      return;
    }
    timer = setTimeout(() => {
      cb();
      timer = null;
    }, 2000);
  }
}
function onLoadingProgressConsoleLog(receivedLength, contentLength) {
  if (!contentLength) {return;}
  const percentage = receivedLength / contentLength;
  const text = (percentage * 100 + "").substring(0, 4) + " % of " + bytesToSizeWinLike(contentLength);

  const needReset = percentage === 1;
  const callback = () => {
    console.log(text);
  };
  logDebouncer(callback, needReset);
}

</script>

<style>
.warning {
  color: red;
}
</style>
