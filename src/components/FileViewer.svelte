<script lang="ts">
  export let imageFile: FileList;
  export let alt: string;

  let imageElement: HTMLImageElement;
  let width = 0;
  let height = 0;

  const onload = () => {
    width = imageElement.naturalWidth;
    height = imageElement.naturalHeight;
  };
</script>

<div class="divider divider-start">ファイルプレビュー</div>
<div class="gird-cols-1 grid md:grid-cols-2">
  <img
    {alt}
    src={URL.createObjectURL(imageFile[0])}
    bind:this={imageElement}
    on:load={onload}
  />
  <table class="table w-full h-fit">
    <tbody>
      <tr>
        <th>ファイル名</th>
        <td>{imageFile[0].name}</td>
      </tr>
      <tr>
        <th>ファイルタイプ</th>
        <td>{imageFile[0].type}</td>
      </tr>
      <tr>
        <th>ファイルサイズ</th>
        <td>{imageFile[0].size  / 1000.0} KB</td>
      </tr>
      <tr>
        <th>幅</th>
        <td>{width}px</td>
      </tr>
      <tr>
        <th>高さ</th>
        <td>{height}px</td>
      </tr>
    </tbody>
  </table>
</div>
