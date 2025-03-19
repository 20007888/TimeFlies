<template>
  <div class="finnegans-wake-container">
    <div class="content-area">
      <div class="text-container">
        <audio ref="audioPlayer" style="display: none"></audio>
        <div v-if="isLoading" class="loading">Loading content...</div>
        <div v-else class="content-text">
          <p
            v-for="(paragraph, pIndex) in currentText"
            :key="pIndex"
            class="paragraph"
            @mouseup="handleTextSelection"
            @mousemove="updateFloatingButtonPosition"
          >
            <span
              v-for="(word, wIndex) in splitIntoParts(paragraph)"
              :key="wIndex"
              class="word"
              @mouseover="showAnnotation(word)"
              @mouseout="hideAnnotation"
              :class="{ 'has-annotation': hasAnnotation(word) }"
            >
              {{ word }}
            </span>
          </p>
        </div>
      </div>

      <!-- Floating play button -->
      <div
        v-if="showFloatingButton"
        class="floating-play-button"
        :style="{
          left: floatingButtonPosition.x + 'px',
          top: floatingButtonPosition.y + 'px',
        }"
      >
        <button
          @click="playSelectedText"
          :disabled="isPlaying"
          class="play-button mini"
        >
          {{ isPlaying ? "Playing..." : "Play" }}
        </button>
        <div class="selected-text-preview">
          {{ truncateText(selectedText, 20) }}
        </div>
      </div>

      <div class="annotation-panel" v-if="currentAnnotation">
        <div class="annotation-content">
          <h3>Annotation</h3>
          <p>{{ currentAnnotation }}</p>
        </div>
      </div>
    </div>

    <!-- Controls moved to bottom -->
    <div class="bottom-controls">
  

      <div class="pagination-controls">
        <button @click="previousPage" :disabled="currentPage <= 1">
          Previous
        </button>
        <span>{{ currentPage }} / {{ totalPages }}</span>
        <button @click="nextPage" :disabled="currentPage >= totalPages">
          Next
        </button>
      </div>

      <div class="text-controls">
        <button
          @click="parseCurrentText"
          :disabled="isProcessing || isLoading"
          class="parse-button"
        >
          {{ isProcessing ? "Processing..." : "Detect Puns" }}
        </button>
      </div>

    </div>
  </div>
</template>

<script>
export default {
  name: "FinnegansWake",
  data() {
    return {
      totalPages: 0,
      currentPage: 1,
      currentText: [],
      currentAnnotation: null,
      dynamicAnnotations: {}, // Store dynamically parsed annotations
      isPlaying: false,
      isProcessing: false,
      isLoading: true,
      audioQueue: [],
      currentAudioIndex: 0,
      eventSource: null,
      API_KEY: "app-qdaCYDbNT6dTD2hufQTTPGgS",
      PARSE_API_KEY: "app-FLOptCRlLAYGKY0G4cCB7KGu",
      selectedText: "",
      debugInfo: [],
      showFloatingButton: false,
      floatingButtonPosition: {
        x: 0,
        y: 0,
      },
      csvData: [], // Store CSV data
      pageSize: 1, // Number of pages to display per view
    };
  },
  methods: {
    async loadCsvData() {
      try {
        this.isLoading = true;
        const response = await fetch("/data/fw_by_page.csv");
        const text = await response.text();

        // Using a simpler method to parse CSV
        // First split by line
        const lines = text.split("\n");

        // Skip the header row
        const contentLines = lines.slice(1);

        // Create a regex to match Page and the following content
        const pageRegex = /^Page\s+(\d+),(.*)$/;
        const pages = [];

        for (let i = 0; i < contentLines.length; i++) {
          const line = contentLines[i].trim();
          if (!line) continue;

          const match = line.match(pageRegex);
          if (match) {
            const pageNumber = parseInt(match[1]);
            // Get the content part - might include quotes
            let content = match[2].trim();

            // If content starts with a quote, it might span multiple lines
            if (content.startsWith('"')) {
              // Remove the starting quote
              content = content.substring(1);

              // If this line doesn't end with a quote, continue reading following lines until finding the end quote
              if (!content.endsWith('"')) {
                let j = i + 1;
                while (j < contentLines.length) {
                  const nextLine = contentLines[j].trim();
                  content += "\n" + nextLine;

                  // If found the end quote, end the loop
                  if (nextLine.endsWith('"')) {
                    i = j; // Update the outer loop index
                    break;
                  }
                  j++;
                }
              }

              // Remove the ending quote
              if (content.endsWith('"')) {
                content = content.substring(0, content.length - 1);
              }
            }

            pages.push({
              page: pageNumber,
              content: content,
            });
          }
        }

        this.csvData = pages;
        this.totalPages = Math.ceil(this.csvData.length / this.pageSize);

        // Load the first page
        await this.loadPage(1);
      } catch (error) {
        console.error("Failed to load CSV data:", error);
        alert("Failed to load data, please try again later");
      } finally {
        this.isLoading = false;
      }
    },
    async loadPage(pageNum) {
      if (pageNum < 1 || pageNum > this.totalPages) return;

      this.isLoading = true;
      this.currentPage = pageNum;

      // Calculate the current page data range
      const startIndex = (pageNum - 1) * this.pageSize;
      const endIndex = Math.min(
        startIndex + this.pageSize,
        this.csvData.length
      );

      // Get the current page data
      const pageData = this.csvData.slice(startIndex, endIndex);

      // Convert the content to paragraph array
      this.currentText = pageData
        .map((item) => {
          // Split into paragraphs by empty lines
          const paragraphs = item.content
            .split(/\n\s*\n/) // Match a newline followed by any whitespace and another newline
            .map((p) => p.trim())
            .filter((p) => p);

          return paragraphs;
        })
        .flat();

      this.isLoading = false;
    },
    previousPage() {
      if (this.currentPage > 1) {
        this.loadPage(this.currentPage - 1);
      }
    },
    nextPage() {
      if (this.currentPage < this.totalPages) {
        this.loadPage(this.currentPage + 1);
      }
    },
    splitIntoParts(text) {
      return text.split(" ");
    },
    showAnnotation(word) {
      const wordLower = word.toLowerCase().replace(/[,.;:!?()]/g, "");

      // 1. First try exact matching
      if (this.dynamicAnnotations.hasOwnProperty(wordLower)) {
        this.currentAnnotation = this.dynamicAnnotations[wordLower];
        return;
      }

      // 2. Check if it's part of a multi-word phrase
      const keys = Object.keys(this.dynamicAnnotations);

      // Look for multi-word phrases containing the current word
      for (const phrase of keys) {
        const phraseParts = phrase.toLowerCase().split(/\s+/);
        if (phraseParts.includes(wordLower) && phraseParts.length > 1) {
          this.currentAnnotation = this.dynamicAnnotations[phrase];
          return;
        }
      }

      // 3. For longer words (>=4 characters), check if it's part of an annotation phrase
      if (wordLower.length >= 4) {
        for (const phrase of keys) {
          const phraseLower = phrase.toLowerCase();
          if (phraseLower.split(/\s+/).includes(wordLower)) {
            this.currentAnnotation = this.dynamicAnnotations[phrase];
            return;
          }
        }
      }

      // If no match, don't show annotation
      this.currentAnnotation = null;
    },
    hideAnnotation() {
      this.currentAnnotation = null;
    },
    hasAnnotation(word) {
      const wordLower = word.toLowerCase().replace(/[,.;:!?()]/g, "");

      // For common short words, must match exactly to return true
      const commonWords = [
        "a",
        "an",
        "the",
        "and",
        "or",
        "but",
        "in",
        "on",
        "at",
        "by",
        "to",
        "of",
        "for",
        "with",
        "from",
      ];
      if (commonWords.includes(wordLower)) {
        return this.dynamicAnnotations.hasOwnProperty(wordLower);
      }

      // 1. First try exact matching
      if (this.dynamicAnnotations.hasOwnProperty(wordLower)) {
        return true;
      }

      // 2. Try to find if a word in annotations is part of the current word (current word is longer than the annotation word)
      const keys = Object.keys(this.dynamicAnnotations);

      // Look for multi-word phrase matches
      // For example, if annotation has "Howth Castle", and current word is "Howth", it should be marked as having annotation
      for (const phrase of keys) {
        const phraseParts = phrase.toLowerCase().split(/\s+/);
        if (phraseParts.includes(wordLower) && phraseParts.length > 1) {
          return true;
        }
      }

      // For short words (less than 4 characters), must match exactly
      if (wordLower.length < 4) {
        return false;
      }

      // 3. For longer words, if it's part of a phrase in annotations and longer than 4 characters, mark it
      for (const phrase of keys) {
        const phraseLower = phrase.toLowerCase();
        // Only when the current word is a longer word (>=4 characters) and is a complete word in an annotation phrase
        if (phraseLower.split(/\s+/).includes(wordLower)) {
          return true;
        }
      }

      return false;
    },
    handleTextSelection(event) {
      const selection = window.getSelection();
      const selectedText = selection.toString().trim();

      if (selectedText) {
        this.selectedText = selectedText;
        this.showFloatingButton = true;

        // Get the position of selected text
        const range = selection.getRangeAt(0);
        const rect = range.getBoundingClientRect();

        // Calculate button position
        const scrollX = window.scrollX || window.pageXOffset;
        const scrollY = window.scrollY || window.pageYOffset;

        this.floatingButtonPosition = {
          x: rect.right + scrollX + 10, // 10px to the right of selected text
          y: rect.top + scrollY - 10, // Offset 10px upward
        };
      } else {
        this.showFloatingButton = false;
      }
    },
    updateFloatingButtonPosition(event) {
      if (!this.showFloatingButton) return;

      // Check if mouse is far from button
      const buttonElement = document.querySelector(".floating-play-button");
      if (buttonElement) {
        const rect = buttonElement.getBoundingClientRect();
        const distance = Math.sqrt(
          Math.pow(event.clientX - rect.left, 2) +
            Math.pow(event.clientY - rect.top, 2)
        );

        // If mouse is more than 200px away from button, hide it
        if (distance > 200) {
          this.showFloatingButton = false;
        }
      }
    },
    truncateText(text, maxLength) {
      if (text.length <= maxLength) return text;
      return text.substring(0, maxLength) + "...";
    },
    async playSelectedText() {
      if (this.isPlaying || !this.selectedText) return;

      this.isPlaying = true;
      this.audioQueue = [];
      this.currentAudioIndex = 0;

      try {
        // Process selected text, remove line breaks
        const processedText = this.selectedText
          .replace(/\n/g, " ")
          .replace(/\s+/g, " ")
          .trim();

        // First parse the text
        const parseResult = await this.parseText(processedText);

        // If parsing is successful, use the parsed text for further processing
        // Make sure we extract the actual text from the result object
        let textToProcess = processedText;
        if (parseResult) {
          if (typeof parseResult === "object") {
            if (parseResult.text) {
              textToProcess = parseResult.text;
            } else if (
              parseResult.answer &&
              typeof parseResult.answer === "string"
            ) {
              textToProcess = parseResult.answer;
            }
          } else if (typeof parseResult === "string") {
            textToProcess = parseResult;
          }
        }

        console.log("Text to process for audio:", textToProcess);

        // First check if proxy server is running
        try {
          const healthCheck = await fetch("http://localhost:3001/health");
          if (!healthCheck.ok) {
            throw new Error("Proxy server not responding");
          }
        } catch (error) {
          throw new Error(
            "Cannot connect to proxy server, please ensure it's running"
          );
        }

        const response = await fetch(
          "http://localhost:3001/api/workflows/run",
          {
            method: "POST",
            headers: {
              Authorization: `Bearer ${this.API_KEY}`,
              "Content-Type": "application/json",
            },
            body: JSON.stringify({
              inputs: {
                input: textToProcess,
              },
              response_mode: "streaming",
              user: "user-" + Date.now(),
            }),
          }
        );

        if (!response.ok) {
          const errorData = await response.json();
          throw new Error(
            `API request failed: ${errorData.message || response.statusText}`
          );
        }

        const reader = response.body.getReader();
        const decoder = new TextDecoder();
        let buffer = "";

        while (true) {
          const { done, value } = await reader.read();
          if (done) break;

          buffer += decoder.decode(value, { stream: true });
          const lines = buffer.split("\n");

          buffer = lines.pop() || "";

          for (const line of lines) {
            if (!line.trim()) continue;

            if (!line.startsWith("data: ")) continue;

            try {
              const jsonStr = line.slice(6);
              const data = JSON.parse(jsonStr);

              // Handle WAV audio URL in node_finished event
              if (
                data.event === "node_finished" &&
                data.data?.files?.length > 0
              ) {
                const audioFile = data.data.files[0];
                if (audioFile.mime_type === "audio/x-wav" && audioFile.url) {
                  this.playAudioUrl(audioFile.url);
                }
              }
              // Handle base64 audio in tts_message event
              else if (data.event === "tts_message" && data.audio) {
                this.audioQueue.push(data.audio);
                if (this.audioQueue.length === 1) {
                  this.playNextAudio();
                }
              }
            } catch (error) {
              console.error(
                "Failed to parse event data:",
                error,
                "Raw data:",
                line
              );
            }
          }
        }

        // Process remaining data in buffer
        if (buffer.trim()) {
          try {
            const jsonStr = buffer.slice(6);
            const data = JSON.parse(jsonStr);
            if (
              data.event === "node_finished" &&
              data.data?.files?.length > 0
            ) {
              const audioFile = data.data.files[0];
              if (audioFile.mime_type === "audio/x-wav" && audioFile.url) {
                this.playAudioUrl(audioFile.url);
              }
            } else if (data.event === "tts_message" && data.audio) {
              this.audioQueue.push(data.audio);
              if (this.audioQueue.length === 1) {
                this.playNextAudio();
              }
            }
          } catch (error) {
            console.error("Failed to parse final event data:", error);
          }
        }
      } catch (error) {
        console.error("Playback failed:", error);
        alert(error.message || "Playback failed, please try again later");
      } finally {
        this.isPlaying = false;
      }
    },
    playNextAudio() {
      if (this.currentAudioIndex >= this.audioQueue.length) {
        this.isPlaying = false;
        return;
      }

      const audio = this.audioQueue[this.currentAudioIndex];
      const audioBlob = this.base64ToBlob(audio);
      const audioUrl = URL.createObjectURL(audioBlob);

      this.$refs.audioPlayer.src = audioUrl;
      this.$refs.audioPlayer.play();

      this.$refs.audioPlayer.onended = () => {
        URL.revokeObjectURL(audioUrl);
        this.currentAudioIndex++;
        this.playNextAudio();
      };
    },
    base64ToBlob(base64) {
      const byteCharacters = atob(base64);
      const byteNumbers = new Array(byteCharacters.length);

      for (let i = 0; i < byteCharacters.length; i++) {
        byteNumbers[i] = byteCharacters.charCodeAt(i);
      }

      const byteArray = new Uint8Array(byteNumbers);
      return new Blob([byteArray], { type: "audio/mp3" });
    },
    // Method to play audio URL
    playAudioUrl(url) {
      this.$refs.audioPlayer.src = url;
      this.$refs.audioPlayer.play().catch((error) => {
        console.error("Failed to play audio URL:", error);
      });
    },
    // Text parsing method
    async parseText(text) {
      try {
        console.log("Parsing text:", text);

        const response = await fetch(
          "http://localhost:3001/api/workflows/run",
          {
            method: "POST",
            headers: {
              Authorization: `Bearer ${this.PARSE_API_KEY}`,
              "Content-Type": "application/json",
            },
            body: JSON.stringify({
              inputs: {
                pun_detection: text,
              },
              response_mode: "blocking",
              conversation_id: "",
              user: "user-" + Date.now(),
            }),
          }
        );

        if (!response.ok) {
          const errorData = await response.json();
          console.error("Parse API response error:", errorData);
          throw new Error(
            `Parse API request failed: ${
              errorData.message || response.statusText
            }`
          );
        }

        const data = await response.json();
        console.log("Complete parse API response:", data);

        // Check the structure of returned data
        if (data.data?.outputs?.text) {
          try {
            // Original returned text content
            const rawText = data.data.outputs.text;
            console.log("Original API returned text:", rawText);

            // Process Markdown code block format
            let jsonString = rawText;

            // Remove Markdown code block format (if present)
            if (rawText.includes("```")) {
              // Extract content between ```json and ```
              const match = rawText.match(/```(?:json)?\n([\s\S]*?)\n```/);
              if (match && match[1]) {
                jsonString = match[1];
              }
            }

            console.log("Processed JSON for parsing:", jsonString);

            // Parse JSON string
            const parsedOutput = JSON.parse(jsonString);
            console.log("Parsed text annotation data:", parsedOutput);
            return {
              answer: parsedOutput,
              text: text,
              conversation_id: data.task_id,
              metadata: {
                task_id: data.task_id,
                workflow_run_id: data.workflow_run_id,
                elapsed_time: data.data.elapsed_time,
                total_tokens: data.data.total_tokens,
              },
            };
          } catch (parseError) {
            console.warn(
              "JSON parsing error, returning original text:",
              parseError
            );
            console.log("Original response text:", data.data.outputs.text);
            return {
              answer: data.data.outputs.text,
              text: text,
              conversation_id: data.task_id,
            };
          }
        } else {
          console.error("API response data format error:", data);
          throw new Error("API returned incorrect data format");
        }
      } catch (error) {
        console.error("Text parsing failed:", error);
        return null; // If parsing fails, return null
      }
    },
    async parseCurrentText() {
      if (this.isProcessing) return;

      this.isProcessing = true;
      this.dynamicAnnotations = {}; // Clear previous annotations

      try {
        // Get all text from current page
        const fullText = this.currentText.join(" ");
        // Replace line breaks with spaces
        const processedText = fullText
          .replace(/\n/g, " ")
          .replace(/\s+/g, " ")
          .trim();

        console.log("Text to parse:", processedText);

        // Call parse API
        const parseResult = await this.parseText(processedText);

        console.log("Parse result:", parseResult);

        if (parseResult) {
          // Process returned annotation data
          if (parseResult.answer && typeof parseResult.answer === "object") {
            // Add object key-value pairs directly to dynamic annotations
            console.log("Adding annotation data to dynamicAnnotations");
            Object.entries(parseResult.answer).forEach(
              ([word, explanation]) => {
                this.dynamicAnnotations[word.toLowerCase()] = explanation;
              }
            );
            console.log(
              "Parsing complete, current annotation data:",
              this.dynamicAnnotations
            );
          }
        } else {
          throw new Error("Text parsing failed");
        }
      } catch (error) {
        console.error("Parsing failed:", error);
        alert(error.message || "Parsing failed, please try again later");
      } finally {
        this.isProcessing = false;
      }
    },
  },
  mounted() {
    // Add handler to hide button when clicking empty space
    document.addEventListener("click", (event) => {
      if (
        !event.target.closest(".floating-play-button") &&
        !window.getSelection().toString().trim()
      ) {
        this.showFloatingButton = false;
      }
    });

    // Load CSV data
    this.loadCsvData();
  },
  beforeDestroy() {
    document.removeEventListener("click", this.hideFloatingButton);
  },
};
</script>

<style scoped>
.finnegans-wake-container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 1rem 2rem;
  font-family: "Times New Roman", serif;
  display: flex;
  flex-direction: column;
  min-height: auto;
  height: auto;
}

.content-area {
  display: grid;
  grid-template-columns: 3fr 1fr;
  gap: 2rem;
  flex-grow: 0;
  margin-bottom: 1.5rem;
  min-height: 0;
  height: auto;
  overflow: visible;
  width: 1200px;
}

.content-text {
  min-height: 100px;
}

.bottom-controls {
  padding-top: 1rem;
  border-top: 1px solid #eaeaea;
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-wrap: wrap;
  gap: 0.5rem;

}

.pagination-controls {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  margin-left: 300px;
}

.text-controls {
  display: flex;
  justify-content: flex-start;
  margin-right: 100px;

}

.pagination-controls button,
.parse-button {
  padding: 0.5rem 1.25rem;
  border: none;
  cursor: pointer;
  border-radius: 4px;
  transition: all 0.3s ease;
  font-size: 0.95rem;
}

.pagination-controls button {
  background-color: #4a4a4a;
  color: white;
}

.parse-button {
  background-color: #4caf50;
  color: white;
}

.pagination-controls button:hover:not(:disabled) {
  background-color: #333;
}

.parse-button:hover:not(:disabled) {
  background-color: #45a049;
}

.pagination-controls button:disabled,
.parse-button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}

.pagination-controls span {
  padding: 0 0.25rem;
  font-size: 1.05rem;
  font-weight: bold;
}

.text-container {
  font-size: 1.2rem;
  line-height: 1.8;
  background-color: #fefefe;
  padding: 1.5rem;
  border-radius: 8px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.05);
  min-height: auto;
  height: auto;
  overflow-y: auto;
}

.loading {
  padding: 2rem;
  text-align: center;
  color: #666;
  font-style: italic;
}

.paragraph {
  margin-bottom: 1.5rem;
}

.paragraph:last-child {
  margin-bottom: 0;
}

.word {
  display: inline-block;
  margin: 0 2px;
  padding: 2px 4px;
  border-radius: 3px;
  cursor: pointer;
}

.word.has-annotation {
  background-color: #fff3d4;
}

.word:hover {
  background-color: #ffe4b5;
}

.annotation-panel {
  background-color: #f8f8f8;
  padding: 1.5rem;
  border-radius: 8px;
  position: sticky;
  top: 2rem;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.05);
  max-height: calc(100vh - 4rem);
  overflow-y: auto;
}

.annotation-content {
  font-size: 1rem;
}

.annotation-content h3 {
  margin-top: 0;
  color: #333;
  margin-bottom: 1rem;
}

.floating-play-button {
  position: absolute;
  z-index: 1000;
  background-color: white;
  border-radius: 8px;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.15);
  padding: 8px;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 4px;
  animation: fadeIn 0.2s ease-out;
}

.play-button.mini {
  font-size: 0.9rem;
  padding: 4px 12px;
  min-width: 60px;
  background-color: #4a69bd;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.3s ease;
}

.play-button.mini:hover:not(:disabled) {
  background-color: #3b5998;
}

.play-button.mini:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}

.selected-text-preview {
  font-size: 0.8rem;
  color: #666;
  max-width: 150px;
  text-overflow: ellipsis;
  overflow: hidden;
  white-space: nowrap;
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(-5px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

/* 响应式调整 */
@media (max-width: 768px) {
  .content-area {
    grid-template-columns: 1fr;
  }

  .bottom-controls {
    flex-direction: column;
    align-items: stretch;
    gap: 1rem;
  }

  .pagination-controls {
    justify-content: center;
  }

  .text-controls {
    justify-content: center;
  }

  .finnegans-wake-container {
    padding: 1rem;
  }

  .text-container {
    padding: 1rem;
  }
}
</style> 