<script setup lang="ts">
import { ref, onMounted, watch } from "vue";
import { useToast } from "primevue/usetoast";
import { showToast } from "../utils";

interface UploadedImage {
  file: File;
  url: string;
  width: number;
  height: number;
}

const uploadedImages = ref<UploadedImage[]>([]);
const stitchedImageUrl = ref<string>("");
const isStitching = ref(false);
const canvasRef = ref<HTMLCanvasElement | null>(null);

const layoutOptions = ref(["Horizontal", "Vertical"]);
const layoutSelected = ref("Horizontal");

const backgroundOptions = ref(["Transparent", "Color"]);
const backgroundSelected = ref("Transparent");
const backgroundColor = ref("#ffffff");

const adjustSizeOptions = ref(["None", "Min", "Max"]);
const adjustSizeSelected = ref("None");

// Debug watcher for color picker
watch(backgroundColor, (newValue) => {
  console.log("Background color changed:", newValue);
});

const toast = useToast();

const onFileSelect = (event: any) => {
  const files = event.files;
  if (files && files.length > 0) {
    // Filter for image files
    const imageFiles = Array.from(files as File[]).filter((file: File) =>
      file.type.startsWith("image/")
    );

    if (imageFiles.length === 0) {
      toast.add({
        severity: "error",
        summary: "Invalid Files",
        detail: "Please select image files only.",
        life: 3000,
      });
      return;
    }

    let loadRemaining = imageFiles.length;

    // Process each image file
    imageFiles.forEach((file: File) => {
      const reader = new FileReader();
      reader.onload = (e) => {
        const img = new Image();
        img.onload = () => {
          const imageData: UploadedImage = {
            file,
            url: e.target?.result as string,
            width: img.width,
            height: img.height,
          };
          uploadedImages.value.push(imageData);
          loadRemaining--;
          if (loadRemaining === 0) {
            stitchImages();

            showToast.success(
              toast,
              "Files Uploaded",
              `${imageFiles.length} image(s) uploaded successfully.`
            );
          }
        };
        img.src = e.target?.result as string;
      };
      reader.readAsDataURL(file);
    });
  }
};

const onFileClear = () => {
  uploadedImages.value = [];
  stitchedImageUrl.value = "";
  toast.add({
    severity: "info",
    summary: "Files Cleared",
    detail: "All uploaded images have been cleared.",
    life: 3000,
  });
};

const stitchImages = async () => {
  if (uploadedImages.value.length < 2) {
    toast.add({
      severity: "error",
      summary: "Insufficient Images",
      detail: "Please upload at least 2 images to stitch.",
      life: 3000,
    });
    return;
  }

  isStitching.value = true;

  try {
    // Create canvas for stitching
    const canvas = document.createElement("canvas");
    const ctx = canvas.getContext("2d");

    if (!ctx) {
      throw new Error("Failed to get canvas context");
    }

    // Load all images first
    const images = await Promise.all(
      uploadedImages.value.map(async (imgData) => {
        return new Promise<HTMLImageElement>((resolve) => {
          const img = new Image();
          img.onload = () => resolve(img);
          img.src = imgData.url;
        });
      })
    );

    // Calculate adjusted dimensions based on size adjustment option
    let adjustedImages: (HTMLImageElement | HTMLCanvasElement)[] = images;

    if (adjustSizeSelected.value !== "None") {
      adjustedImages = images.map((img) => {
        const canvas = document.createElement("canvas");
        const ctx = canvas.getContext("2d");

        if (!ctx) return img;

        let targetWidth = img.width;
        let targetHeight = img.height;

        if (layoutSelected.value === "Horizontal") {
          // For horizontal layout, adjust height
          if (adjustSizeSelected.value === "Min") {
            const minHeight = Math.min(...images.map((img) => img.height));
            targetHeight = minHeight;
          } else if (adjustSizeSelected.value === "Max") {
            const maxHeight = Math.max(...images.map((img) => img.height));
            targetHeight = maxHeight;
          }
        } else {
          // For vertical layout, adjust width
          if (adjustSizeSelected.value === "Min") {
            const minWidth = Math.min(...images.map((img) => img.width));
            targetWidth = minWidth;
          } else if (adjustSizeSelected.value === "Max") {
            const maxWidth = Math.max(...images.map((img) => img.width));
            targetWidth = maxWidth;
          }
        }

        // Calculate new dimensions maintaining aspect ratio
        const aspectRatio = img.width / img.height;
        let newWidth = targetWidth;
        let newHeight = targetHeight;

        if (layoutSelected.value === "Horizontal") {
          // For horizontal, maintain aspect ratio by adjusting width
          newWidth = targetHeight * aspectRatio;
        } else {
          // For vertical, maintain aspect ratio by adjusting height
          newHeight = targetWidth / aspectRatio;
        }

        // Create resized image
        canvas.width = newWidth;
        canvas.height = newHeight;
        ctx.drawImage(img, 0, 0, newWidth, newHeight);

        return canvas;
      });
    }

    // Calculate canvas dimensions based on adjusted images
    let totalWidth = 0;
    let totalHeight = 0;

    if (layoutSelected.value === "Horizontal") {
      // Calculate total width and max height from adjusted images
      totalWidth = adjustedImages.reduce((sum, img) => sum + img.width, 0);
      totalHeight = Math.max(...adjustedImages.map((img) => img.height));
    } else {
      // Calculate max width and total height from adjusted images
      totalWidth = Math.max(...adjustedImages.map((img) => img.width));
      totalHeight = adjustedImages.reduce((sum, img) => sum + img.height, 0);
    }

    // Set canvas dimensions
    canvas.width = totalWidth;
    canvas.height = totalHeight;

    // Fill with background color or transparent
    if (backgroundSelected.value === "Color") {
      console.log("Background color selected:", backgroundColor.value);
      // Ensure the color is in the correct format
      let color = backgroundColor.value || "#ffffff";

      // Ensure it's a valid hex color
      if (typeof color === "string" && !color.startsWith("#")) {
        color = `#${color}`;
      }

      console.log("Final color to apply:", color);
      ctx.fillStyle = color;
      ctx.fillRect(0, 0, totalWidth, totalHeight);
    }

    // Draw images
    let currentX = 0;
    let currentY = 0;

    for (let i = 0; i < adjustedImages.length; i++) {
      const img = adjustedImages[i];

      if (layoutSelected.value === "Horizontal") {
        ctx.drawImage(img, currentX, 0);
        currentX += img.width;
      } else {
        ctx.drawImage(img, 0, currentY);
        currentY += img.height;
      }
    }

    // Convert canvas to data URL with proper transparency handling
    if (backgroundSelected.value === "Transparent") {
      // For transparent background, we need to ensure the canvas is properly transparent
      stitchedImageUrl.value = canvas.toDataURL("image/png");
    } else {
      // For colored background, use PNG to preserve quality
      stitchedImageUrl.value = canvas.toDataURL("image/png");
    }
  } catch (error) {
    console.error("Error stitching images:", error);
    showToast.error(
      toast,
      "Stitching Failed",
      "Failed to stitch images: " + error
    );
  } finally {
    isStitching.value = false;
  }
};

const downloadStitchedImage = () => {
  if (!stitchedImageUrl.value) {
    showToast.error(
      toast,
      "No Image",
      "No stitched image available to download."
    );
    return;
  }

  const link = document.createElement("a");
  link.href = stitchedImageUrl.value;
  link.download = `stitched-image-${Date.now()}.png`;
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);

  showToast.success(
    toast,
    "Download Started",
    "Stitched image download has started."
  );
};

const moveImageUp = (index: number) => {
  if (index > 0) {
    const temp = uploadedImages.value[index];
    uploadedImages.value[index] = uploadedImages.value[index - 1];
    uploadedImages.value[index - 1] = temp;
    stitchedImageUrl.value = "";
    stitchImages();
  }
};

const moveImageDown = (index: number) => {
  if (index < uploadedImages.value.length - 1) {
    const temp = uploadedImages.value[index];
    uploadedImages.value[index] = uploadedImages.value[index + 1];
    uploadedImages.value[index + 1] = temp;
    stitchedImageUrl.value = "";
    stitchImages();
  }
};

const removeImage = (index: number) => {
  uploadedImages.value.splice(index, 1);
  stitchedImageUrl.value = "";

  // Reset background to transparent if no images left
  if (uploadedImages.value.length === 0) {
    backgroundSelected.value = "Transparent";
    adjustSizeSelected.value = "None";
  }

  stitchImages();
};

// on color change, stitch images
watch(backgroundColor, () => {
  console.log("backgroundColor changed:", backgroundColor.value);
});
</script>

<template>
  <div class="container">
    <Toast />

    <!-- File Upload Section -->
    <Card>
      <template #title>Upload Images</template>
      <template #content>
        <div class="upload-section">
          <FileUpload
            name="images[]"
            :multiple="true"
            accept="image/*"
            :maxFileSize="10000000"
            @select="onFileSelect"
            @clear="onFileClear"
            :auto="true"
            chooseLabel="Choose Images"
            cancelLabel="Clear"
            :showCancelButton="true"
            :showUploadButton="false"
            :customUpload="true"
            class="file-upload"
          />
        </div>
      </template>
    </Card>

    <!-- Uploaded Images Preview -->
    <Card v-if="uploadedImages.length > 0">
      <template #title>Uploaded Images ({{ uploadedImages.length }})</template>
      <template #content>
        <div class="images-preview">
          <div
            v-for="(image, index) in uploadedImages"
            :key="index"
            class="image-item"
          >
            <img :src="image.url" :alt="`Image ${index + 1}`" />
            <div class="image-info">
              <span>{{ image.file.name }}</span>
              <span>{{ image.width }} × {{ image.height }}</span>
            </div>
            <div class="image-controls">
              <ButtonGroup>
                <Button
                  icon="pi pi-chevron-up"
                  severity="secondary"
                  size="small"
                  @click="moveImageUp(index)"
                  :disabled="index === 0"
                  title="Move Up"
                />
                <Button
                  icon="pi pi-chevron-down"
                  severity="secondary"
                  size="small"
                  @click="moveImageDown(index)"
                  :disabled="index === uploadedImages.length - 1"
                  title="Move Down"
                />
              </ButtonGroup>
              <Button
                icon="pi pi-trash"
                severity="danger"
                size="small"
                @click="removeImage(index)"
                title="Remove"
              />
            </div>
          </div>
        </div>
      </template>
    </Card>

    <!-- Stitching Controls -->
    <Card v-if="uploadedImages.length >= 2">
      <template #title>Image Stitching</template>
      <template #content>
        <div class="stitching-controls">
          <div class="layout-selection">
            <label>Layout:</label>
            <SelectButton
              @change="stitchImages"
              v-model="layoutSelected"
              :options="layoutOptions"
              class="layout-buttons"
            />
          </div>

          <div class="background-selection">
            <label>Background:</label>
            <SelectButton
              @change="stitchImages"
              v-model="backgroundSelected"
              :options="backgroundOptions"
              class="background-buttons"
            />
            <div
              v-if="backgroundSelected === 'Color'"
              class="current-color-display"
              :style="{ backgroundColor: '#' + backgroundColor }"
            >
              Current: {{ "#" + backgroundColor }}
            </div>
          </div>

          <div
            v-if="backgroundSelected === 'Color'"
            class="color-picker-section"
          >
            <label>Background Color:</label>
            <div class="color-picker-wrapper">
              <ColorPicker
                v-model="backgroundColor"
                class="color-picker"
                format="hex"
                @change="stitchImages"
              />
            </div>
          </div>

          <div class="adjust-size-selection">
            <label>Adjust Size:</label>
            <SelectButton
              v-model="adjustSizeSelected"
              :options="adjustSizeOptions"
              class="adjust-size-buttons"
              @change="stitchImages"
            />
          </div>

          <div class="stitched-preview">
            <img
              :src="stitchedImageUrl"
              alt="Stitched Image"
              class="stitched-image"
            />
          </div>

          <div class="action-buttons">
            <Button
              v-if="stitchedImageUrl"
              label="Download"
              icon="pi pi-download"
              severity="success"
              @click="downloadStitchedImage"
              class="download-btn"
            />
          </div>
        </div>
      </template>
    </Card>
  </div>
</template>

<style scoped lang="less">
.container {
  display: flex;
  flex-direction: column;
  gap: 20px;
  padding: 20px;
  max-width: 1200px;
  margin: 0 auto;
}

.upload-section {
  .file-upload {
    width: 100%;
  }
}

.images-preview {
  display: flex;
  flex-direction: column;
  gap: 12px;
  margin-top: 16px;

  .image-item {
    position: relative;
    border-radius: 8px;
    overflow: hidden;
    background: #8f8f8fa8;
    display: flex;
    align-items: center;
    padding: 4px;
    padding-right: 18px;
    gap: 12px;

    img {
      width: 60px;
      height: 60px;
      object-fit: cover;
      display: block;
      border-radius: 4px;
      flex-shrink: 0;
    }

    .image-info {
      display: flex;
      gap: 8px;
      flex: 1;
      font-size: 0.875rem;
      color: #666;

      span {
        display: block;
        margin-bottom: 2px;

        &:first-child {
          font-weight: 500;
          color: #333;
        }
      }
    }

    .image-controls {
      // position: absolute;
      display: flex;
      gap: 4px;
      top: 8px;
      right: 8px;
      border-radius: 4px;
      backdrop-filter: blur(4px);
    }
  }
}

.stitching-controls {
  display: flex;
  flex-direction: column;
  gap: 16px;

  .layout-selection,
  .background-selection,
  .adjust-size-selection {
    display: flex;
    align-items: center;
    gap: 12px;

    label {
      font-weight: 500;
      color: #333;
      min-width: 80px;
    }

    .layout-buttons,
    .background-buttons,
    .adjust-size-buttons {
      flex: 1;
    }

    .current-color-display {
      padding: 4px 8px;
      border-radius: 4px;
      font-size: 0.75rem;
      color: white;
      text-shadow: 1px 1px 1px rgba(0, 0, 0, 0.5);
      margin-top: 4px;
    }
  }

  .color-picker-section {
    display: flex;
    align-items: center;
    gap: 12px;

    label {
      font-weight: 500;
      color: #333;
      min-width: 120px;
    }

    .color-picker-wrapper {
      display: flex;
      align-items: center;
      gap: 8px;
      flex: 1;
    }

    .color-picker {
      flex: 1;
      max-width: 200px;
    }

    .color-preview {
      width: 32px;
      height: 32px;
      border: 2px solid #e0e0e0;
      border-radius: 4px;
      cursor: pointer;
    }
  }

  .action-buttons {
    display: flex;
    gap: 12px;
    flex-wrap: wrap;

    .stitch-btn {
      flex: 1;
      min-width: 150px;
    }

    .download-btn {
      flex: 1;
      min-width: 150px;
    }
  }
}

.stitched-preview {
  display: flex;
  justify-content: center;
  margin-top: 16px;

  .stitched-image {
    max-width: 100%;
    max-height: 600px;
    border: 1px solid #e0e0e0;
    border-radius: 8px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  }
}

// Responsive design
@media (max-width: 768px) {
  .container {
    padding: 16px;
  }

  .images-preview {
    gap: 8px;
  }

  .image-item {
    padding: 8px;
    gap: 8px;

    img {
      width: 60px;
      height: 60px;
    }
  }

  .stitching-controls {
    .layout-selection,
    .background-selection,
    .adjust-size-selection {
      flex-direction: column;
      align-items: flex-start;
      gap: 8px;
    }

    .color-picker-section {
      flex-direction: column;
      align-items: flex-start;
      gap: 8px;

      .color-picker-wrapper {
        width: 100%;
        flex-direction: column;
        align-items: flex-start;
        gap: 8px;
      }

      .color-picker {
        max-width: 100%;
      }
    }

    .action-buttons {
      flex-direction: column;

      .stitch-btn,
      .download-btn {
        flex: none;
        width: 100%;
      }
    }
  }
}
</style>
