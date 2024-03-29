<template>
  <div class="hello">

    <h1 style="font-weight:bold; color:rgb(46, 24, 145);">L.U.N.A.L.A</h1>

    <p style="margin-bottom:40px">
      The Logographic Uncluttered Natural Aquisition Language Achiever
    </p>

    <b-dropdown text="Origin Selection" 
      ref="dropdown" 
      class="m-2"
      variant="info"
      >
      <div class="dd_div">
        <b-dropdown-form>
          <b-form-group v-slot="{ ariaDescribedby }">
            <b-form-checkbox-group
              v-model="selected_origin"
              :options="origins"
              :aria-describedby="ariaDescribedby"
              name="flavour-1a"
              @change="updateChecklist()"
            ></b-form-checkbox-group>
          </b-form-group>
        </b-dropdown-form>
      </div>
    </b-dropdown>

    <!-- Difficulty selection field -->
    <b-form-group v-slot="{ ariaDescribedby }">
      <b-form-checkbox-group
        v-model="selected_diff"
        :options="difficulties"
        :aria-describedby="ariaDescribedby"
        name="flavour-1a"
        @change="updateChecklist()"
      ></b-form-checkbox-group>
    </b-form-group>

    <!-- Card amount selection field -->
    <b-form-group v-slot="{ ariaDescribedby }">
      <b-form-checkbox-group
        v-model="selected_cards"
        :options="numCards"
        :aria-describedby="ariaDescribedby"
        name="flavour-1a"
        @change="updateChecklist()"
      ></b-form-checkbox-group>
    </b-form-group>

    <!-- Generate Cards button -->
    <b-button class="button" 
      style="margin-bottom:40px; background-color:rgb(211, 209, 1); color:rgb(46, 24, 145);" 
      @click="generateCards()"
      v-b-tooltip.hover.left title="Generate a new shuffled batch of cards with the above selection fields."
      data-toggle="tooltip"
      >Generate Cards</b-button>

    <!-- Update Difficulties toggle button -->
    <b-button class="button" 
      style="margin-bottom:40px; background-color:rgb(211, 209, 1); color:rgb(46, 24, 145);" 
      @click="setDifficulties()"
      v-b-tooltip.hover.right title="Toggle on/off whether clicking a card changes the language or the difficulty."
      data-toggle="tooltip"
      >Update Difficulties</b-button>
    
    <div v-if="!dataLoaded">
      <div class="spinner-border text-info" role="status">
        <span class="sr-only">Loading...</span>
      </div>
      <p>Loading Cards...</p>
    </div>

    <!-- Card Deck: only display after data collected from flask app -->
    <div v-if="dataLoaded" class="card_div">
      <b-card-group deck v-for="row in cardGrid" :key="cardGrid.indexOf(row)" class="cards"> 
        <b-card v-for="item in row" 
          :key="item" 
          class="card" 
          text-variant="white" 
          :header='getDifficulty(vocab_file["Difficulty"][item])' 
          header-bg-variant="info"
          >
          <b-card-text class="card_text">{{getText(item)}}</b-card-text>
          <button class="stretched-link" 
            style="height:0px; background-color:rgb(46, 24, 145); border-style:none" 
            @click="cycleCard(item)"></button>
        </b-card>
      </b-card-group>
      <b-button class="button" style="margin-bottom:40px; background-color:rgb(211, 209, 1); color:rgb(46, 24, 145);" @click="getCards(true)">Next Cards</b-button>
    </div>

    <!-- Save Changes button -->
    <div>
      <b-button class="button" style="margin-bottom:40px; margin-top:40px; background-color:rgb(23, 162, 184)" v-b-modal.auth-modal @click="replace=false">Save Changes</b-button> 
      <b-button class="button" style="margin-bottom:40px; margin-top:40px" href="https://lunala-api.herokuapp.com/getCSV" target="_blank">Download Flashcards</b-button> 
    </div>

    <!-- User form input field -->
    <b-form-file
      class="file_in"
      v-model="file_input"
      :state="Boolean(file_input)"
      accept=".csv"
      placeholder="Choose a replacement CSV here..."
      drop-placeholder="Drop file here..."
      @change="loadCSV"
    ></b-form-file>
    <div>
      <b-button class="button" style="margin-bottom:40px" v-b-modal.auth-modal @click="replace=true">Submit</b-button> 
    </div>
    
    <!-- Popup auth modal for overwrites -->
    <b-modal id="auth-modal" hide-footer title="Save Changes">
      <div>
        <b-form @submit.prevent="replace ? replaceCSV() : saveChanges()">
          <b-form-group label="Username:">
            <b-form-input type="text" v-model="user_in" required placeholder="Enter username"></b-form-input>
          </b-form-group>
          <b-form-group label="Password:">
            <b-form-input type="password" v-model="pass_in" required placeholder="Enter password"></b-form-input>
          </b-form-group>
          <b-button type="submit" variant="info">Submit</b-button>
          <br/>
          <br/>
          <b-alert show>Be sure you want to save these changes! They will overwrite the current flashcard file!</b-alert>
        </b-form>
      </div>
    </b-modal>
    
  </div>
</template>

<script>
import axios from 'axios'

const NUM_HEADERS = 6;    // Number of headers the file validator checks for in user input files
// const CARDS_PER_ROW = 7;  // Number of cards to display per row 

export default {
  name: 'Main',
  data() {
    return {
      CARDS_PER_ROW: 7,   // Number of cards per row 
      vocab_file: [],     // Not actually a file; JSON object of the cards to display
      file_input: [],     // The actual input file
      offset: 0,          // The paging offset 
      origins: [],        // The set of possible origins for each term
      dupMap: {},
      numSet: [],         // List of the order in which cards are displayed
      cardGrid: [],       // The cardgrid to display (2D array)
      cardStates: {},     // The text state of the cards
      dataLoaded: false,  // Boolean: determine if init data retrieved
      difficulties: ["New", "Easy", "Medium", "Hard"],  // Difficulty selection options
      numCards: [],       // Card amount selection options
      selected_origin: [],
      selected_diff: ["New", "Easy", "Medium", "Hard"], // Initial difficulties selected
      selected_cards: [],    // Initial card selection
      difficultyMode: false, // Initial difficulty mode toggle 
      replace: false,        // Whether to replace the CSV or save changes to it
      user_in: "",      // User username
      pass_in: "",      // User password
      userAuth: false,  // Whether or not user is authorized
    }
  },
  methods: {

    /*
      getVocab method: 
      Retrieves the vocabulary CSV file from the flask API as a JSON object.
      Alerts the user on failure. 
    */
    async getVocab() {
      // let path = 'http://localHost:5000/getVocab' // local route
      let path = 'https://lunala-api.herokuapp.com/getVocab'
      await axios.get(path)
        .then((res) => {
          this.vocab_file = res.data;
          console.log('Vocab data:', this.vocab_file);
        })
        .catch(() => {
          alert("API connection failed; please input your own CSV.");
        })
    },

    /*
      findDuplicates method:
      Finds duplicate terms and discludes them from the deck 
    */
    findDuplicates() {
      Object.keys(this.vocab_file['Kanji']).forEach(index => {
        if (!(this.vocab_file['Kanji'][index] in this.dupMap)) {
          this.dupMap[this.vocab_file['Kanji'][index]] = [];
        }
        this.dupMap[this.vocab_file['Kanji'][index]].push(index);
      });
      // console.log("Duplicate Map:", this.dupMap);
    },

    /*
      updateOrigins method:
      Determins the complete set of origins for all fetched terms
    */
    updateOrigins() {
      this.origins = new Set();
      Object.keys(this.vocab_file['Origin']).forEach(index => {
        this.origins.add(this.vocab_file['Origin'][index]);
      });
      this.origins = Array.from(this.origins);
      this.origins.forEach(origin => {
        this.selected_origin.push(origin);
      });
      console.log("Origins:", this.origins);
    },

    /*
      loadCSV method:
      Upon user file input, reads the file as text and ensures its headers
      are properly formatted (see checkData).
    */
    loadCSV(ev) {
      const newFile = ev.target.files[0]
      const reader = new FileReader();
      reader.readAsText(newFile);
      reader.onload = e => this.checkData(e.target.result);
    },

    /*
      checkData method: 
      Accepts a string formatted CSV file (file parameter).
      Ensures the headers are properly formatted; if not, discards the 
      user input file. 
    */
    checkData(file) {
      let fileAsLines = file.split("\n");
      let headers = fileAsLines[0].split(",");
      headers[headers.length-1] = headers[headers.length-1].replace(/^[\r\n]+|\.|[\r\n]+$/g, ""); // Remove final carriage return on last item 
      console.log('Headers:', headers)
      if (headers.length != NUM_HEADERS) {
        alert("CSV File not correctly formatted. \n\n\
              The CSV should contain columns in any order of: \n\
              Kanji, Hiragana, English, Origin, Difficulty, and Category")
        this.file_input = [];
      }
    },

    /*
      replaceCSV method:
      Sends the user input file to the flask API and replaces the old file. 
      Resets default selection fields. 
      Generates a new set of cards from the new file. 
    */
    async replaceCSV() {
      this.userAuth = false;
      this.authenticate().then(async() => {
        if (this.userAuth) {
          console.log("FILE", this.file_input)
          if (this.file_input != []) {
            let formData = new FormData();
            formData.append('file', this.file_input);
            // let path = 'http://localHost:5000/replace' // local route
            let path = 'https://lunala-api.herokuapp.com/replace'
            await axios.post(path, formData, { 
              headers: {
                'Content-Type': 'multipart/form-data' 
              }
            }).then(() => {
              alert('New CSV successfully sent.');
            }).catch((error) => {
              alert('Authenticated, but file failed to send.');
              console.log("File send error:", error)
            });

            // Make new card deck 
            this.dataLoaded = false;
            await this.getVocab();
            this.findDuplicates();
            this.updateOrigins();
            this.selected_diff = ["New", "Easy", "Medium", "Hard"];
            this.selected_cards = [36];
            this.shuffleCards();
            this.generateCards(0);
            this.dataLoaded = true;
          }
        }
        else {
          alert("Authentication failed. Please try again!");
        }
      })
    },

    /*
      shuffleCards method:
      Filters cards to display based on the user selection. 
      Shuffles all the cards in the card list. 
    */
    shuffleCards() {

      // Fisher-Yates algorithm 
      let shuffle = function(arr) {
        let currIndex = arr.length, randomIndex;
        while (currIndex !== 0) {
          randomIndex = Math.floor(Math.random() * currIndex);
          currIndex--;
          [arr[currIndex], arr[randomIndex]] = [arr[randomIndex], arr[currIndex]];
        }
        return arr;
      }
      let numCards = Object.keys(this.vocab_file['Kanji']).length; // Number of cards in the deck 
      this.numSet = [];
      let disclude = [];
      let collisions = [];
      for (let i = 0; i < numCards; i++) {
        if (this.selected_diff.includes(this.getDifficulty(this.vocab_file['Difficulty'][i]))) {
          if (this.selected_origin.includes(this.vocab_file['Origin'][i])) {
            if (!disclude.includes(this.vocab_file['Kanji'][i])) {
              this.numSet.push(i);
              disclude.push(this.vocab_file['Kanji'][i]);
            }
            else {
              collisions.push(i + ": " + this.vocab_file['Kanji'][i])
            }
          }  
        }
      }
      this.numSet = shuffle(this.numSet);
      //console.log(this.numSet.length, this.numSet)
      console.log("Collisions:", collisions);
    },

    /*
      generateCards method:
      Generates a new set of cards to display. 
    */
    generateCards() {
      this.offset = 0;
      this.shuffleCards();
      this.getCards(true);
    },

    /*
      getCards method:
      Retrieves the next batch of cards to display to the user through the use of paging.
    */
    getCards(nextPage) {
      if (this.offset <= Object.keys(this.vocab_file['Kanji']).length) {
        window.scrollTo(0, 400); // Scroll to top of page every time next batch retrieved
        let breakEarly = false;  // Boolean: if cards to display run out 
        this.cardGrid = [];     
        let numRows = this.selected_cards[0]/this.CARDS_PER_ROW;
        for (let i = 0; i < numRows; i++) {
          if (breakEarly) {
            break;
          }
          let currRow = [];
          for (let j = 0; j < this.CARDS_PER_ROW; j++) {

            // If possible, add the next card from the shuffled array to the grid and init to the logographic text 
            try {
              currRow.push(this.numSet[this.CARDS_PER_ROW*i+j+this.offset]);
              this.cardStates[this.numSet[this.CARDS_PER_ROW*i+j+this.offset].toString()] = 0;
            }
            catch (error) {
              console.log("Not enough cards in the deck.");
              breakEarly = true;
              currRow.pop(); // Remove the "undefined" element that was added 
              break;
            }
          }
          this.cardGrid.push(currRow); // Add the completed row to the 2D card grid 
          // console.log(this.cardGrid)
        }
        // Do not page on window resize handler
        if (nextPage) {
          this.offset += this.selected_cards[0]; // Increment the paging offset for the next batch 
        }
        // console.log(this.cardGrid);
        // console.log(this.cardStates);
      }
    },

    /*
      getDifficulty method:
      Maps integer representations of difficulty to strings,
      or string representations of difficulty to integers. 
      Returns the interprested value of the input. 
    */
    getDifficulty(num) {
      if (num === 0) {
        return "New"
      }
      if (num === 1) {
        return "Easy"
      }
      if (num === 2) {
        return "Medium"
      }
      if (num === 3) {
        return "Hard"
      }
      if (num === "New") {
        return 1
      }
      if (num === "Easy") {
        return 1
      }
      if (num === "Easy") {
        return 2
      }
      if (num === "Easy") {
        return 3
      }
    },

    /*
      getText method: 
      Accepts an integer row index of the original CSV (item).
      Returns the text to display on the card to the user 
    */
    getText(item) {
      if (this.cardStates[item] === 0) {
        return this.vocab_file["Kanji"][item];
      }
      if (this.cardStates[item] === 1) {
        return this.vocab_file["Hiragana"][item];
      }
      return this.vocab_file["English"][item];
    },

    /*
      cycleCard method:
      Accepts an integer row index of the original CSV (item). 
      From user clicks on a card, cycles through the text display 
      for the term or the difficulty. 
    */
    cycleCard(item) {
      
      // Cycle text state
      if (!this.difficultyMode) {
        this.cardStates[item] = this.cardStates[item] + 1; 
        if (this.cardStates[item] > 2) {
          this.cardStates[item] = 0;
        }
        // console.log(this.cardStates[item]);
      }

      // Cycle difficlty state
      else {
        let termsToUpdate = this.dupMap[this.vocab_file["Kanji"][item]]; // Update same terms across sources
        console.log("Updating:", termsToUpdate);
        let consistencyTerm = termsToUpdate[0]; // Ensure all terms across sources have same difficulty
        let newDiff = -1;
        if (this.vocab_file["Difficulty"][consistencyTerm] < 3) {
            newDiff = this.vocab_file["Difficulty"][consistencyTerm] + 1;
          }
          else {
            newDiff = 0;
          }
        termsToUpdate.forEach(term => {
          this.vocab_file["Difficulty"][term] = newDiff;
          console.log(term, this.vocab_file["Kanji"][term], this.vocab_file["Difficulty"][term])
        });
      }
      this.$forceUpdate();
    },

    /*
      setDifficulties method: 
      Toggles the difficulty selection mode. 
    */
    setDifficulties() {
      this.difficultyMode = !this.difficultyMode;
    },

    /*
      updateChecklist method:
      Updates the selection checklist (ensures the user can only select one of the card_number options).
    */
    updateChecklist() {
      let currCard = this.selected_cards[this.selected_cards.length-1];

      // Only allow user to select a single card_number option
      this.selected_cards = [];
      this.selected_cards.push(currCard);
      // console.log(this.selected_diff)
      // console.log(this.selected_cards)
    },

    /*
      authenticate method: 
      Ensures that the user entered the correct credentials. 
      Store the credentials.txt file in a folder called credentials under the lunala directory 
    */
    async authenticate() {
      // let path = 'http://localHost:5000/getAuth' // local route
      let credentials = {}
      let path = 'https://lunala-api.herokuapp.com/getAuth'
      await axios.get(path)
        .then((res) => {
          credentials = res.data;
          console.log('Credentials retrieved.');
        })
        .catch(() => {
          alert("API connection failed; please try again.");
        })
      Object.keys(credentials).forEach(user => {
        if (this.user_in === user) {
          if (this.pass_in === credentials[this.user_in]) {
            this.userAuth = true;
            return true;
          }
          return false;
        }
      })
      return false;
    },

    /*
      saveChanges method: 
      Sends the updated cards to the flask API app. Overwrites the original vocabulary file
      if authentication passed. 
    */
    async saveChanges() {
      this.userAuth = false;
      this.authenticate().then(() => {
        if (this.userAuth) {
          // let path = 'http://localHost:5000/saveCSV'// local path
          let path = 'https://lunala-api.herokuapp.com/saveCSV'
          axios.post(path, this.vocab_file)
            .then(() => {
              alert("Changes successfully saved.");
            })
            .catch((error) => {
              alert("Authenticated, but file failed to send");
              console.log("File send error:", error);
            })
        }
        else {
          alert("Authentication failed! Changes unsaved.")
        }
        console.log("Sent flash cards:", this.vocab_file);
      })
    },

    /*
      setCardsPerRow method:
      Listener method: determines number of cards to display based on the screen size
      Dynamically updates as the screen size is updated.
      Also updates the card count options.
    */
    setCardsPerRow() {
      this.numCards = [];
      let screenWidth = Math.max(
        document.documentElement.clientWidth,
        window.innerWidth || 0
      );
      // console.log("Width", screenWidth);
      if (screenWidth <= 300) {
        this.CARDS_PER_ROW = 1;
      } 
      else if (screenWidth <= 560) {
        this.CARDS_PER_ROW = 2;
      }
      else if (screenWidth <= 768) {
        this.CARDS_PER_ROW = 3;
      }
      else if (screenWidth <= 1000) {
        this.CARDS_PER_ROW = 4;
      } 
      else if (screenWidth <= 1200) {
        this.CARDS_PER_ROW = 5;
      } 
      else if (screenWidth <= 1400) {
        this.CARDS_PER_ROW = 6;
      }
      else if (screenWidth <= 1600) {
        this.CARDS_PER_ROW = 7;
      }
      else {
        this.CARDS_PER_ROW = 8;
      }

      // Determine card number options as 2, 5, 10, 40, and 67 rows
      this.numCards.push(this.CARDS_PER_ROW * 2); 
      this.numCards.push(this.CARDS_PER_ROW * 5);
      this.numCards.push(this.CARDS_PER_ROW * 10);
      this.numCards.push(this.CARDS_PER_ROW * 40);
      this.numCards.push(this.CARDS_PER_ROW * 67);

      this.selected_cards = [this.CARDS_PER_ROW * 5];

      // Force the screen to update and get the cards
      this.$forceUpdate();
      if (this.dataLoaded) {
        this.getCards(false);
      }
    }
  }, 

  /*
    beforeMount:
    Ensure the vocabulary file is retrieved and the first set of shuffled cards is displayed 
    before executing anything else. 
  */
  async beforeMount() {
    this.setCardsPerRow();
    await this.getVocab();
    this.findDuplicates();
    this.updateOrigins();
    this.shuffleCards();
    this.generateCards(0);
    this.dataLoaded = true; 
  },

  /*
    Checks for window resizes and updates the cards accordings
  */
  created() {
    window.addEventListener("resize", this.setCardsPerRow);
  },
  destroyed() {
    window.removeEventListener("resize", this.setCardsPerRow);
  },

}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
.button {
  margin-left: 10px;
  margin-right: 10px;
  margin-top: 20px;
  margin-bottom: 40px;
  background-color:rgb(46, 24, 145);
  border:none;
} 
.file_in {
  width: 20%;
  text-align: left;
  margin-top: 30px;
}
.dd_div {
  overflow-y: auto; 
  height: 200px;
  width: 250px;
}
.cards {
  margin: auto;
  width: 80%;
  margin-bottom: 20px;
}
.card {
  background-color:rgb(46, 24, 145);
}
.card_text {
  font-size: 23px;
  color:rgb(255, 255, 255);
}
h1 {
  margin: 20px 0 10px;
}
h3 {
  margin: 40px 0 0;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
</style>
