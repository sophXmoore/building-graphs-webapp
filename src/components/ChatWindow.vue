<template>
  <div style="height: 100vh;" class="d-flex flex-column">
    <!-- Logo -->
    <div class="d-flex justify-left align-center">
      <img src="@/assets/logo.png" alt="Logo" 
        style="height: 50px;" />
    </div>
    

    <!-- CHAT MESSAGES -->
    <div style="overflow-y: auto; flex-grow: 1;" class="pa-2">
      <div v-for="message in allMessages" 
        class="d-flex" :class="message.type === 'userMessage' ? 'justify-end' : 'justify-start'">
        <!-- User Message -->
        <span v-if="message.type == 'userMessage'" 
          class="pa-2 my-2 rounded-lg"
          style="background-color: lightgray;">
          {{ message.message }}
        </span>
        <!-- AI Response -->
        <span v-else v-html="message.message" class="ml-6"></span>
    </div>
    </div>

    <!-- Textarea and send button -->
    <div class="pa-2" style="border-top: 1px solid #ddd;">
      <v-textarea 
        v-model="inputMessage"
        hide-details
        variant="outlined"
        auto-grow
        label="Ask your graph questions here"
      />
      <div class="d-flex justify-end mt-2">
        <v-btn
          @click="sendRequest"
          :loading="sendLoading"
          size="small"
          icon
          variant="text"
          class="bg-black"
        >
          <v-icon>mdi-arrow-up</v-icon>
        </v-btn>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import { OpenAI } from 'openai'
import {useStore} from '@/stores/store'
import neo4j from 'neo4j-driver'

const store = useStore()

//Initialize Variables
const inputMessage = ref('')
const allMessages = ref([])
var sendLoading = ref(false)

//Create OpenAI Connection
const openai = new OpenAI({
  apiKey: import.meta.env.VITE_OPENAI_API_KEY,
  dangerouslyAllowBrowser: true
})

//Create Neo4j Connection
const neo4jDriver = neo4j.driver(
  import.meta.env.VITE_NEO4J_URI,
  neo4j.auth.basic(
    import.meta.env.VITE_NEO4J_USER,
    import.meta.env.VITE_NEO4J_PASSWORD
  )
)

//Function to execute Neo4j Cypher queries
async function executeNeo4jQuery(cypherQuery, parameters = {}) {
  const session = neo4jDriver.session()
  try {
    const result = await session.run(cypherQuery, parameters)
    return result.records.map(record => record.toObject())
  } catch (error) {
    console.error('Neo4j query error:', error)
    throw error
  } finally {
    await session.close()
  }
}

async function sendRequest() {
  //Step 1: Show that the request is being sent
  sendLoading = true;

  //Step 2: Add User's message to the chat log
  allMessages.value.push({
    message: inputMessage.value,
    type: 'userMessage'
  })

  //Step 3: Set up prompt
  const conversation = [
    {
      role: "system",
      content: `
        You are an AI that answers questions about a Neo4j graph database.

        The database structure is:
        - Node label: Nodes
        - Node properties: id, name, description, label, objectType, centroid, category
        - centroid is defined as an array [x, y, z]
        - Relationships: Edges (undirected adjacency between spaces)

        When you need data, request it by replying ONLY in this form:

        QUERY:
        <cypher query>

        Rules for queries:
        - Do NOT use backticks or code fences.
        - Return only the Cypher query after "QUERY:" with no explanation.

        When you have enough data, respond normally (do not use QUERY:).
        If a previous query returned an error, correct the query and try again.
        `
    },
    { role: "user", content: inputMessage.value }
  ];

  try {
    //Iterative create and execute cypher queries.
    let maxIterations = 8
    for (let step = 0; step < maxIterations; step++) {  
      let response;
      
      try {
        response = await openai.chat.completions.create({
          model: "gpt-4.1",  
          messages: conversation,
        });
      } catch (err) {
        console.error("OpenAI request failed:", err);
        throw err;
      }

      console.log('OpenAI Response', response)
      const reply = response.choices[0].message.content.trim();

      if (reply.startsWith("QUERY:")) {
        const cypherQuery = reply.replace("QUERY:", "").trim();

        try {
          const result = await executeNeo4jQuery(cypherQuery);

          // Add result to conversation
          conversation.push(
            { role: "assistant", content: reply },
            { role: "user", content: `RESULT:\n${JSON.stringify(result, null, 2)}` }
          );

        } catch (cypherError) {
          console.warn("Cypher Error:", cypherError);

          // Add any errors to the conversation and start another iteration
          conversation.push(
            { role: "assistant", content: reply },
            { role: "user", content: `ERROR:\n${cypherError.message}\nPlease fix the query and try again.` }
          );

          continue; // Try another iteration
        }

        continue; // Continue the loop to allow further queries
      }

      // Push final answer to chat log 
      allMessages.value.push({ message: reply, type: "openAIResponse" });
      break;
    }
  } catch (fatalError) {
    allMessages.value.push({
      message: `Error: ${fatalError.message}`,
      type: "openAIResponse"
    });
  }

  inputMessage.value = "";
  sendLoading = false;
}

</script>

<style scoped>
</style>