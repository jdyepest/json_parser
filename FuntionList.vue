<template>
  <div class="container">
    <h1 class="title">Paste JSON Schema</h1>
    <!-- Textarea to allow JSON input -->
    <textarea v-model="jsonInput" placeholder="Paste your JSON here" rows="10" class="textarea"></textarea>
    <button @click="parseJson" class="button parse-button">
      Parse
    </button>

    <div v-if="parseError" class="error">
      <p>{{ parseError }}</p>
    </div>

    <!-- Display the parsed functions and their arguments -->
    <div v-if="functions.length">
      <h2 class="section-title">Function Schemas</h2>
      <div v-for="(fn, index) in functions" :key="fn.function.name" class="function-item">
        <!-- Toggle editable fields based on `isEditing` state -->
        <div v-if="!fn.isEditing">
          <h3 class="function-name">{{ fn.function.name }}</h3>
          <p class="function-description">{{ fn.function.description }}</p>
        </div>
        <div v-else>
          <!-- Editable function name -->
          <label class="label">Function Name:</label>
          <input v-model="fn.function.name" class="input" />

          <!-- Editable description -->
          <label class="label">Description:</label>
          <textarea v-model="fn.function.description" class="textarea"></textarea>
        </div>

        <!-- Parameters Section -->
        <h4 class="section-subtitle">Parameters:</h4>
        <div v-for="(param, key) in fn.function.parameters.properties" :key="key" class="parameter-item">
          <strong class="label">{{ key }}:</strong>
          <div v-if="!fn.isEditing">
            <p>{{ param.description }}</p>
            <!-- Show type in view mode -->
            <div>
              <strong>Type:</strong> {{ param.type }}
            </div>
            <!-- Show format in view mode -->
            <div v-if="param.format">
              <strong>Format:</strong> {{ param.format }}
            </div>
            <!-- Show enum values in view mode -->
            <div v-if="param.enum">
              <strong>Enum Values:</strong> {{ param.enum.join(', ') }}
            </div>

            <!-- Handle array type -->
            <div v-if="param.type === 'array'">
              <strong>Items Type:</strong> {{ param.items.type }}
              <div v-if="param.items.format">
                <strong>Items Format:</strong> {{ param.items.format }}
              </div>
              <div v-if="param.items.enum">
                <strong>Items Enum:</strong> {{ param.items.enum.join(', ') }}
              </div>
            </div>
          </div>

          <div v-else>
            <!-- Editable parameter description -->
            <label class="label">Description:</label>
            <input v-model="param.description" class="input" />

            <!-- Editable type as a dropdown -->
            <label class="label">Type:</label>
            <select v-model="param.type" class="input">
              <option v-for="typeOption in typeOptions" :key="typeOption" :value="typeOption">
                {{ typeOption }}
              </option>
            </select>

            <!-- Editable format as a dropdown with suggestions -->
            <label class="label">Format:</label>
            <input list="formatOptions" v-model="param.format" class="input" />
            <datalist id="formatOptions">
              <option v-for="formatOption in formatOptions" :key="formatOption" :value="formatOption">
                {{ formatOption }}
              </option>
            </datalist>

            <!-- Editable field for enum values -->
            <div v-if="param.enum">
              <label class="label">Enum (comma-separated):</label>
              <input v-model="param.enum" @input="updateEnum(param)" class="input" />
            </div>

            <!-- Handle array type -->
            <div v-if="param.type === 'array'">
              <label class="label">Array Items Type:</label>
              <select v-model="param.items.type" class="input">
                <option v-for="typeOption in typeOptions" :key="typeOption" :value="typeOption">
                  {{ typeOption }}
                </option>
              </select>

              <!-- Editable items format -->
              <label class="label">Array Items Format:</label>
              <input list="formatOptions" v-model="param.items.format" class="input" />

              <!-- Editable enum for array items -->
              <div v-if="param.items.enum">
                <label class="label">Array Items Enum (comma-separated):</label>
                <input v-model="param.items.enum" @input="updateEnum(param.items)" class="input" />
              </div>
            </div>
          </div>
        </div>

        <!-- Required fields section -->
        <h4 class="section-subtitle">Required Fields:</h4>
        <div v-if="!fn.isEditing">
          <p>{{ fn.function.parameters.required.join(', ') }}</p>
        </div>
        <div v-else>
          <div v-for="(param, key) in fn.function.parameters.properties" :key="key" class="required-item">
            <input type="checkbox" :id="key" :value="key" v-model="fn.function.parameters.required" class="checkbox" />
            <label :for="key">{{ key }}</label>
          </div>
        </div>

        <!-- Buttons for Edit, Save, and Remove -->
        <div class="button-group">
          <button v-if="!fn.isEditing" @click="toggleEdit(index)" class="button edit-button">
            Edit
          </button>
          <button v-else @click="toggleEdit(index)" class="button save-button">
            Save
          </button>
          <button @click="removeFunction(index)" class="button remove-button">
            Remove Function
          </button>
        </div>

        <!-- Line between functions -->
        <hr class="divider" />
      </div>
    </div>

    <!-- Button to export JSON -->
    <div v-if="functions.length" class="export-buttons">
      <button @click="exportJsonToClipboard" class="button export-button">
        Copy JSON to Clipboard
      </button>
      <button @click="exportJsonToNewTab" class="button export-button">
        Open JSON in New Tab
      </button>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      jsonInput: '',  // JSON input from the user
      functions: [],  // Parsed functions
      parseError: null,  // Error handling for JSON parsing
      typeOptions: ['string', 'number', 'integer', 'object', 'array', 'boolean', 'null'],  // JSON schema types
      formatOptions: [  // Format suggestions
        'date', 'time', 'date-time', 'duration', 'regex', 'email', 'idn-email',
        'hostname', 'idn-hostname', 'ipv4', 'ipv6', 'json-pointer', 'relative-json-pointer',
        'uri', 'uri-reference', 'uri-template', 'iri', 'iri-reference', 'uuid'
      ]
    };
  },
  methods: {
    // Method to parse the input JSON
    parseJson() {
      try {
        this.functions = JSON.parse(this.jsonInput).map(fn => ({
          ...fn,
          isEditing: false,  // Initialize the `isEditing` flag to false for each function
          function: {
            ...fn.function,
            parameters: {
              ...fn.function.parameters,
              required: fn.function.parameters.required || [],  // Ensure `required` is an array
              properties: Object.fromEntries(
                Object.entries(fn.function.parameters.properties).map(([key, param]) => {
                  const updatedParam = {
                    ...param
                  };

                  if (param.type === 'array') {
                    updatedParam.items = param.items ? { ...param.items } : {};
                  }

                  return [key, updatedParam];
                })
              ),
            },
          },
        }));
        this.parseError = null;  // Clear any previous errors
      } catch (error) {
        this.parseError = "Invalid JSON format. Please check and try again.";  // Handle JSON errors
      }
    },
    // Method to update the enum array based on the input string
    updateEnum(param) {
      if (param.enum && typeof param.enum === 'string') {
        param.enum = param.enum.split(',').map(value => value.trim());  // Convert comma-separated string to array
      }
    },
    // Method to remove the `isEditing` flag from the exported JSON
    cleanJsonData() {
      return this.functions.map(fn => {
        const { isEditing, ...functionData } = fn;
        return functionData;
      });
    },
    // Export the modified JSON to the clipboard
    exportJsonToClipboard() {
      const jsonToCopy = JSON.stringify(this.cleanJsonData(), null, 2);
      navigator.clipboard.writeText(jsonToCopy).then(() => {
        alert('JSON copied to clipboard!');
      });
    },
    // Export the modified JSON and open it in a new tab
    exportJsonToNewTab() {
      const jsonToOpen = JSON.stringify(this.cleanJsonData(), null, 2);
      const newTab = window.open();
      newTab.document.write('<pre>' + jsonToOpen + '</pre>');
    },
    // Method to remove a function and log the function name in the console
    removeFunction(index) {
      const functionName = this.functions[index].function.name;
      console.log(`Removed function: ${functionName}`);  // Log the function name
      this.functions.splice(index, 1);  // Remove the function from the list
    },
    // Method to toggle edit mode for a specific function
    toggleEdit(index) {
      this.functions[index].isEditing = !this.functions[index].isEditing;  // Toggle the `isEditing` flag
    },
  },
};
</script>





<style scoped>
.container {
  width: 100%;
  max-width: 1600px;
  margin: 0 auto;
  padding: 20px;
  font-family: Arial, sans-serif;
}

.title {
  font-size: 2rem;
  text-align: center;
  font-weight: bold;
  margin-bottom: 20px;
}

.textarea {
  width: 100%;
  padding: 10px;
  border: 1px solid #ccc;
  border-radius: 5px;
  margin-bottom: 20px;
}

.input {
  width: 100%;
  padding: 10px;
  border: 1px solid #ccc;
  border-radius: 5px;
  margin-bottom: 20px;
}

.label {
  font-weight: bold;
  margin-bottom: 5px;
  color: #007BFF;
}

.checkbox {
  margin-right: 10px;
}

.button {
  padding: 10px 20px;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  font-weight: bold;
}

.parse-button {
  background-color: #4CAF50;
  color: white;
  margin-bottom: 20px;
}

.edit-button {
  background-color: #FFA500;
  color: white;
}

.save-button {
  background-color: #28A745;
  color: white;
}

.remove-button {
  background-color: #DC3545;
  color: white;
}

.export-buttons {
  margin-top: 40px;
}

.export-button {
  background-color: #007BFF;
  color: white;
  margin-right: 10px;
}

.button-group {
  margin-top: 20px;
  display: flex;
  gap: 10px;
}

.divider {
  margin-top: 30px;
  margin-bottom: 30px;
  border: 0;
  height: 1px;
  background-color: #ccc;
}

.function-item {
  margin-bottom: 40px;
  padding: 20px;
  border: 1px solid #ccc;
  border-radius: 5px;
  background-color: #f9f9f9;
}

.section-title {
  font-size: 1.5rem;
  font-weight: bold;
  margin-bottom: 20px;
}

.section-subtitle {
  font-size: 1.2rem;
  margin-bottom: 10px;
}

.error {
  color: red;
}

.required-item {
  margin-bottom: 10px;
}

.parameter-item {
  margin-bottom: 20px; /* Increase the bottom margin */
}


</style>
