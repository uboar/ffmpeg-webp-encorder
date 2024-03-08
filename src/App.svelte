<script lang="ts">
  import { FFmpeg } from '@ffmpeg/ffmpeg';
  import { onMount } from 'svelte';
  import { toBlobURL } from '@ffmpeg/util';
  import Navbar from './components/Navbar.svelte';
  import FileViewer from './components/FileViewer.svelte';

  const baseURL = 'https://unpkg.com/@ffmpeg/core@0.12.6/dist/esm';
  const ffmpeg = new FFmpeg();

  let loaded = false;
  let ffmpegLogBuffer = '';
  onMount(async () => {
    loaded = await ffmpeg.load({
      coreURL: await toBlobURL(`${baseURL}/ffmpeg-core.js`, 'text/javascript'),
      wasmURL: await toBlobURL(
        `${baseURL}/ffmpeg-core.wasm`,
        'application/wasm',
      ),
    });
    ffmpeg.on('log', (log) => {
      ffmpegLogBuffer += '\n' + log.message;
    });
  });

  const FFmpegArgsTemplates = {
    ロスレス: '-lossless 1',
    ロスレスGIFアニメ: '-lossless 1 -loop 0',
    劣化圧縮: '-crf 1',
    縦幅サイズ指定劣化圧縮: '-vf scale=-1:任意の数字',
  };
  type InputFile = {
    imageFile?: FileList;
    ffmpegArgs: string;
  };

  let inputFiles: InputFile[] = [
    {
      ffmpegArgs: '-lossless 1',
    },
  ];

  const addInputFile = () => {
    inputFiles = [
      ...inputFiles,
      {
        ffmpegArgs: '-lossless 1',
      },
    ];
  };

  const deleteInputFile = (index: number) => {
    inputFiles.splice(index, 1);
    inputFiles = inputFiles;
  };

  type OutputFile = {
    title: string;
    imageFile: FileList;
    ffmpegLog: string;
  };

  let outputFiles: OutputFile[] = [];

  const exec = async (inputFile: InputFile, title: string) => {
    if (inputFile.imageFile != null && inputFile.imageFile.length > 0) {
      await ffmpeg.writeFile(
        inputFile.imageFile[0].name,
        new Uint8Array(await inputFile.imageFile[0].arrayBuffer()),
      );
      await ffmpeg.exec([
        '-i',
        inputFile.imageFile[0].name,
        ...inputFile.ffmpegArgs.split(' '),
        `convert.webp`,
      ]);
      const data = await ffmpeg.readFile('convert.webp');
      const fileList = new DataTransfer();
      fileList.items.add(
        new File(
          [(data as Uint8Array).buffer],
          `${inputFile.imageFile[0].name.split('.').slice(0, -1).join('.')}.webp`,
          { type: 'image/webp' },
        ),
      );
      outputFiles = [
        ...outputFiles,
        {
          title,
          imageFile: fileList.files,
          ffmpegLog: ffmpegLogBuffer,
        },
      ];
      ffmpegLogBuffer = '';
      return true;
    } else {
      return false;
    }
  };

  const deleteOutputFile = (index: number) => {
    outputFiles.splice(index, 1);
    outputFiles = outputFiles;
  };
</script>

<main>
  {#if !loaded}
    <div>
      <span class="loading loading-spinner loading-sm"></span>
      FFmpeg.wasm Loading...
    </div>
  {:else}
    <Navbar></Navbar>
    <div class="grid grid-cols-1 gap-2 p-4 md:grid-cols-2">
      <div class="card card-compact">
        <div class="card-body">
          <div class="card-title text-xl">Input Files</div>
          {#each inputFiles as inputFile, index}
            <div class="collapse collapse-arrow bg-base-300">
              <input type="checkbox" checked />
              <div class="collapse-title text-xl">#{index + 1}</div>
              <div class="collapse-content">
                <label class="form-control w-full">
                  <div class="label">
                    <div class="label-text">入力ファイル</div>
                  </div>
                  <input
                    type="file"
                    bind:files={inputFile.imageFile}
                    class="file-input file-input-bordered"
                  />
                </label>
                <label class="form-control w-full">
                  <div class="label">
                    <div class="label-text">FFmpeg引数</div>
                  </div>
                  <input
                    type="text"
                    bind:value={inputFile.ffmpegArgs}
                    placeholder="e.g.-lossless 1"
                    class="input input-bordered"
                  />
                </label>
                <div class="join mt-4">
                  {#each Object.entries(FFmpegArgsTemplates) as [key, value]}
                    <button
                      class="btn btn-outline join-item"
                      on:click={() => {
                        inputFile.ffmpegArgs = value;
                      }}>{key}</button
                    >
                  {/each}
                </div>
                {#if inputFile.imageFile != null && inputFile.imageFile.length > 0}
                  <FileViewer
                    imageFile={inputFile.imageFile}
                    alt={`#${index + 1}`}
                  ></FileViewer>
                  <button
                    class="btn btn-primary btn-block mt-4"
                    on:click={() => {
                      exec(inputFile, `#${index + 1}`);
                    }}>変換</button
                  >
                {/if}
                <button
                  class="btn btn-outline btn-error btn-sm btn-block mt-4"
                  on:click={() => {
                    deleteInputFile(index);
                  }}>✕</button
                >
              </div>
            </div>
          {/each}
          <button
            class="btn btn-outline btn-primary btn-block mt-4"
            on:click={addInputFile}>＋</button
          >
        </div>
      </div>
      <div class="card card-compact">
        <div class="card-body">
          <div class="card-title text-xl">Output Files</div>
          {#each outputFiles as outputFile, index}
            <div class="collapse collapse-arrow bg-base-300">
              <input type="checkbox" checked />
              <div class="collapse-title text-xl">{outputFile.title}</div>
              <div class="collapse-content">
                <label class="form-control w-full">
                  <div class="label">
                    <div class="label-text">FFmpegログ</div>
                  </div>
                  <textarea
                    class="textarea"
                    readonly
                    bind:value={outputFile.ffmpegLog}
                  ></textarea>
                </label>
                {#if outputFile.imageFile != null && outputFile.imageFile.length > 0}
                  <FileViewer
                    imageFile={outputFile.imageFile}
                    alt={`#${index + 1}`}
                  ></FileViewer>
                  <a
                    href={URL.createObjectURL(outputFile.imageFile[0])}
                    class="btn btn-outline btn-info btn-block mt-4"
                    download={outputFile.imageFile[0].name}
                    target="_blank">ダウンロード</a
                  >
                {/if}
                <button
                  class="btn btn-outline btn-error btn-sm btn-block mt-4"
                  on:click={() => {
                    deleteOutputFile(index);
                  }}>✕</button
                >
              </div>
            </div>
          {/each}
        </div>
      </div>
    </div>
  {/if}
</main>
