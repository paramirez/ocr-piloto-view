<style>
input[type="file"] {
  position: absolute;
  top: -500px;
}

div.file-listing {
  width: 200px;
}

span.remove-file {
  color: red;
  cursor: pointer;
  float: right;
}
.preview {
  max-width: 600px;
}
.buttons {
  padding: 10px;
  max-width: 600px;
  width: 100%;
}
.buttons > * {
  margin: 10px;
  max-width: 500px;
}
.buttons > img {
  width: 100%;
  max-width: 500px;
}
</style>

<template>
  <div class="container">
    <a-spin :spinning="spinning" size="large">
      <a-tabs v-model="activeKey">
        <a-tab-pane tab="Archivo" key="1">
          <div v-if="!files.length">
            <input type="file" id="file" name="file" ref="files" @change="handleFilesUpload">
            <a-button @click="addFiles" icon="upload" type="dashed">Seleccionar Archivo</a-button>
          </div>
          <div v-else class="buttons">
            <a-button @click="removeFile(0)" icon="delete" type="danger">Remover Archivo</a-button>
            <a-button @click="submitFiles('vision')" icon="google" type="dashed">Google Vision</a-button>
            <a-button @click="submitFiles('textract')" icon="amazon" type="dashed">Amazon TextRact</a-button>
            <a-button @click="submitFiles('textract/analyze')" icon="amazon" type="dashed">Amazon TextRact Tables</a-button>
            <img :src="imageData">
          </div>
        </a-tab-pane>
        <a-tab-pane tab="Respuesta" key="2">
          <pre>{{ resultClean }}</pre>
        </a-tab-pane>
        <a-tab-pane tab="Respuesta Limpia" key="3">
          <pre>{{result}}</pre>
        </a-tab-pane>
      </a-tabs>
    </a-spin>
  </div>
</template>

<script>
import axios from "axios";
export default {
  /*
      Defines the data used by the component
    */
  data() {
    return {
      files: [],
      result: "",
      imageData: "",
      resultClean: "",
      activeKey: "1",
      spinning: false
    };
  },

  /*
      Defines the method used by the component
    */
  methods: {
    /*
        Adds a file
      */
    addFiles() {
      this.$refs.files.click();
    },

    /*
        Submits files to the server
      */
    submitFiles(service) {
      /*
          Initialize the form data
        */
      let formData = new FormData();
      this.spinning = true;
      /*
          Iteate over any file sent over appending the files
          to the form data.
        */
      for (var i = 0; i < this.files.length; i++) {
        let file = this.files[i];

        formData.append("file", file);
      }

      /*
          Make the request to the POST /select-files URL
        */
      axios
        .post(
          `http://localhost:3003/${service}`,
          // `https://boiling-thicket-67686.herokuapp.com/${service}`,
          formData,
          {
            headers: {
              "Content-Type": "multipart/form-data"
            }
          }
        )
        .then(res => {
          if (res.data) {
            this.result = res.data;
            if (service === "vision")
              this.resultClean = this.equalize(res.data);
            else if("textract/analyze") this.resultClean = this.equalizeAws(res.data.data);
            else this.resultClean = this.equalizeAws(res.data.data);
            this.activeKey = "2";
          } else {
            this.result = "Sin respuesta";
          }
          this.spinning = false;
        })
        .catch(err => {
          console.log(err);
          this.result = err;
          this.spinning = false;
          this.$message.error("ocurrio un eror");
          this.activeKey = "3";
        });
    },

    /*
        Handles the uploading of files
      */
    handleFilesUpload(event) {
      let uploadedFiles = this.$refs.files.files;

      /*
          Adds the uploaded file to the files array
        */
      for (var i = 0; i < uploadedFiles.length; i++) {
        this.files.push(uploadedFiles[i]);
      }

      // Reference to the DOM input element
      var input = event.target;
      // Ensure that you have a file before attempting to read it
      if (input.files && input.files[0]) {
        // create a new FileReader to read this image and convert to base64 format
        var reader = new FileReader();
        // Define a callback function to run, when FileReader finishes its job
        reader.onload = e => {
          // Note: arrow function used here, so that "this.imageData" refers to the imageData of Vue component
          // Read image as base64 and set to imageData
          this.imageData = e.target.result;
        };
        // Start the reader job - read file as a data url (base64 format)
        reader.readAsDataURL(input.files[0]);
      }
    },

    equalize(result) {
      let finalText = [];
      Object.keys(result).map(key => {
        // if(key === '0') return
        let sumX = 0;
        let sumY = 0;
        result[key].boundingPoly.vertices.map(position => {
          Object.keys(position).map(posKey => {
            if (posKey === "x") {
              sumX += position[posKey];
            } else {
              sumY += position[posKey];
            }
          });
        });
        let x = Math.ceil(sumX / 4);
        let y = Math.ceil(sumY / 4);
        finalText.push({ text: result[key].description, location: { x, y } });
      });
      let answer = {};
      let temp = finalText[0].text.split(/\n/);
      Object.keys(temp).map(key => {
        answer[key] = temp[key];
      });
      return answer;
    },
    equalizeAws(result) {
      const blocks = result.Blocks;
      console.log(result);
      const answer = {};
      if (Array.isArray(blocks)) {
        blocks.forEach((block, index) => {
          if (block.BlockType === "LINE") {
            if (block.Text) {
              answer[`${index}`] = block.Text;
            }
          }
        });
      }
      return answer;
    },
    equalizeAwsAnalyze(result) {
      const blocks = result.Blocks;
      console.log(result);
      const answerKeyValue = {};
      const answer = {};
      if (Array.isArray(blocks)) {
        blocks.forEach((block, index) => {
          if (block.BlockType === "LINE") {
            if (block.Text) {
              answer[`${index}`] = block.Text;
            }
          }
          if (block.BlockType === "KEY_VALUE_SET") {
            if (block.EntityTypes) {
              block.EntityTypes.map(e => {
                if(e === 'KEY') {

                  const values = block.Relationships.filter(i => i.Type === 'VALUE');
                  values.forEach(value => {
                    
                  })
                }
              })
              answer[`${index}`] = block.Text;
            }
          }
        });
      }
      return {
        answer,
        answerKeyValue
      };
    },
    /*
        Removes a select file the user has uploaded
      */
    removeFile(key) {
      this.files.splice(key, 1);
      this.imageData = "";
    }
  }
};
</script>