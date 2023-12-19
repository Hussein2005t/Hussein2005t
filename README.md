ai = {import:ai-text-plugin}
comments = {import:comments-plugin}
fullscreenButton = {import:fullscreen-button-plugin}

storyWritingPrompt
  instruction
    Your task is to write the [storySoFarEl.value.trim() ? "next" : "first"] part of an award-winning story. Don't wrap it up! Just write 4 more paragraphs.
    [whatHappensNextEl.value.trim() ? "IMPORTANT: The first two or three paragraphs that you write MUST be based on this instruction/idea: **"+whatHappensNextEl.value.trim()+"**" : ""]
    # Overall, here's what the story is about:
    [storyOverviewEl.value.trim() || "(Unsure, you decide.)"]
    \n\n
    # Here's what has happened so far:
    [storySoFarEl.value.trim().split("\n\n").slice(0, -1).join("\n\n").trim() || "(Nothing yet.)"] // include all but the last paragraph, since we put the last paragraph in `startWith`
    \n\n
    Write the [storySoFarEl.value.trim() ? "next" : "first"] paragraph of this story based on the above instruction. You are to write four more paragraphs only. No more, no less.
    NEVER try to "wrap up" the story. Avoid repetitive and cliche phrases. NEVER repeat earlier events.
    Your writing should be interesting, authentic, descriptive, natural, engaging, and creative.
    [storySoFarEl.value.trim() ? "" : `Start the story in an unusual and interesting way. Don't use cliche, formulaic, hackneyed phrases. `]Don't be predictable and boring. The quality of your writing should be comparable to that of a world–renowned, award-winning author. Original, subtle, engaging, authentic, grounded, nuanced.
    [whatHappensNextEl.value.trim() ? "IMPORTANT: Remember, the first two or three paragraphs that you write MUST be a creative interpretation of this instruction/idea: **"+whatHappensNextEl.value.trim()+"**" : ""]
    // leave the below line unedited (it joins the above lines into one block of text)
    $output = [this.joinItems("\n")]
  
  startWith = [getStartWithText()] // put last paragraph in `startWith`
  
  // CAUTION: note to self: if you change the stop sequence, ensure that the onChunk new-line removal stuff still makes sense
  stopSequences = [oneParagraphAtATimeCheckbox.checked ? ["\n\n"] : []]
  
  onStart(data) => // runs when the 'regenerate' button is pressed
    storySoFarEl.scrollTop = 99999999;
    generateBtn.disabled = true;
    regenLastBtn.disabled = true;
    deleteLastBtn.disabled = true;
    generateBtn.textContent = "⌛ loading...";
    stopBtn.style.display = "block";
  onChunk(data) =>
    if(data.isFromStartWith) {
      // we don't put the startWith text into the textarea because it's already there
    } else {
      // we're manually adding each chunk of generated text to the "storySoFarEl" text box, rather than using `outputTo`, since `outputTo` would clear all the existing text and only add the *response*, whereas we want to preserve all the existing text, and just add the response to the end. 
      if(oneParagraphAtATimeCheckbox.checked) {
        storySoFarEl.value += data.textChunk.replace(/\n\n$/g, ""); // don't add trailing newlines - to prevent "scroll jank" in onFinish when we trim() the story
      } else {
        storySoFarEl.value += data.textChunk; 
      }
    }
    if(storySoFarEl.scrollTop > (storySoFarEl.scrollHeight - storySoFarEl.offsetHeight)-30) { // <-- if the text box is already scrolled near the end of the text
      storySoFarEl.scrollTop = 99999999; // scroll down to bottom of text box as story streams in
    }
  onFinish(data) =>
    generateBtn.disabled = false;
    regenLastBtn.disabled = false;
    deleteLastBtn.disabled = false;
    generateBtn.textContent = "▶️ next paragraph";
    stopBtn.style.display = "none";
    storySoFarEl.value = storySoFarEl.value.trim(); // remove newlines and spaces from the end of the story
    localStorage.storySoFar = storySoFarEl.value;
    updateButtonsDisplay();

getStartWithText() =>
  let text = storySoFarEl.value.trim().split("\n\n").slice(-1).join("\n\n").trim();
  if(storySoFarEl.value !== "" && /[.?!":»’”—–]$/.test(storySoFarEl.value.trim())) { // if the story textbox isn't empty and it ends with a fullstop, question mark, quote, etc.
    return text+"\n\n"; // then we add a couple of new lines to the end, ready for the next paragraph that's about to be generated
  } else {
    return text;
  }

async continueStory(opts) =>
  window.userClickedStop = false;
  resetRatingButtons();
  
  if(!opts) opts = {};
  if(!opts.continueInline && storySoFarEl.value !== "" && /[.?!":»’”—–]$/.test(storySoFarEl.value.trim())) { // if the story textbox isn't empty and it ends with a fullstop, question mark, quote, etc.
    storySoFarEl.value = storySoFarEl.value.trim() + "\n\n"; // then we add a couple of new lines to the end, ready for the next paragraph that's about to be generated
  }
  if(opts.continueInline) {
    storySoFarEl.value = storySoFarEl.value.trim();
  }
  window.storyTextBeforeLastGeneration = storySoFarEl.value;
  window.lastGenerationStreamObj = ai(storyWritingPrompt); // we put it into a 'global' variable so that we can use it in the 'onclick' of the stop button to stop the text generation
  let data = await window.lastGenerationStreamObj.onFinishPromise;
  
  if(localStorage.generateCount === undefined || isNaN(Number(localStorage.generateCount))) localStorage.generateCount = "0";
  localStorage.generateCount = Number(localStorage.generateCount) + 1;
  updateLastParagraphButtonsDisplayIfNeeded();
  
  if(data.stopReason !== "error" && !window.userClickedStop) {
    enableRatingButtons();
  }

updateLastParagraphButtonsDisplayIfNeeded() =>
  let generateCount = Number(localStorage.generateCount);
  if(generateCount > 40000) {
    rateLastMessageCtn.style.display = "inline-block";
    deleteLastBtn.textContent = "🗑️";
    deleteLastBtn.style.minWidth = "3rem";
    deleteLastBtn.style.marginLeft = "1rem";
    regenLastBtn.textContent = "🔁";
    regenLastBtn.style.minWidth = "3rem";
  }

updateButtonsDisplay() =>
  if(storySoFarEl.value.trim() === "") {
    bottomButtonsCtn.style.display = "none";
  } else {
    bottomButtonsCtn.style.display = "";
    generateBtn.textContent = "▶️ next paragraph";
  }
  
generateWhatHappensNextIdeas() =>
  whatHappensNextSuggestionsCtn.style.display = "";
  generateWhatHappensNextIdeasBtn.disabled = true;
  
  let textSoFar = "";
  let pendingObj = ai({
    instruction: whatHappensNextInstruction.evaluateItem,
    startWith: `Here are 3 different ideas for what could happen next in this story:\n1.`,
    onChunk: (data) => {
      textSoFar += data.textChunk;
      if(!data.isFromStartWith) {
        whatHappensNextSuggestionsCtn.innerHTML = textSoFar.replace(/\n+/g, "\n\n");
      }
    },
    onFinish: () => {
      let existingInstruction = (window.whatHappensNextSuggestionsRegenInstructions || "").trim().replace(/</g, '&lt;').replace(/>/g, '&gt;').replace(/"/g, '&quot;');
      
      let html = textSoFar.trim().split("\n").filter(l => /^[0-9]+\./.test(l.trim())).map(l => l.replace(/^[0-9]+\./g, "").trim()).map(ideaText => {
        let ideaTextEscaped = ideaText.replace(/</g, '&lt;').replace(/>/g, '&gt;').replace(/"/g, '&quot;');
        return `<div style="border:1px solid gray; margin:0.25rem; display:flex; padding:0.25rem; border-radius:3px;">
          <div style="">${ideaTextEscaped}</div>
          <button data-idea="${ideaTextEscaped}" onclick="whatHappensNextEl.value=this.dataset.idea; whatHappensNextSuggestionsCtn.style.display='none';">use</button>
        </div>`;
      }).join("");
      
      html = `<div style="max-height: min(90vh, 600px); overflow: auto ">${html}</div>`;
      html += `<div style="display:flex; margin:0.25rem;"><input value="${existingInstruction}" oninput="window.whatHappensNextSuggestionsRegenInstructions=this.value" placeholder="(Optional) Idea regen instructions." style="flex-grow: 1;"><button style="margin-left: 0.25rem;" onclick="generateWhatHappensNextIdeas()">🔁 regen</button></div>`;
      
      whatHappensNextSuggestionsCtn.innerHTML = html;
      generateWhatHappensNextIdeasBtn.disabled = false;
    },
  });
  whatHappensNextSuggestionsCtn.innerHTML = pendingObj.loadingIndicatorHtml;

whatHappensNextInstruction
  Please write 3 *short* one-sentence, creative ideas for what could happen next in this story.
  [window.whatHappensNextSuggestionsRegenInstructions?.trim() ? `IMPORTANT: Your ideas **MUST** be based on this instruction: ${window.whatHappensNextSuggestionsRegenInstructions}` : ""]
  [""]
  [storySoFarEl.value.trim()]
  [""]
  Again, please write 3 one-sentence ideas. They should be unique, creative, high-level ideas for what could happen next. Just give a few words for each idea.
  Your ideas should be comparable to that of a world–renowned, award-winning author. Original, subtle, realistic, engaging, authentic, grounded, nuanced.
  Each idea must be a SINGLE, *short* sentence.
  [window.whatHappensNextSuggestionsRegenInstructions?.trim() ? `IMPORTANT: Your ideas **MUST** be based on this instruction: ${window.whatHappensNextSuggestionsRegenInstructions}` : ""]
  Follow this template:
  [""]
  1. <a short, **ONE-SENTENCE** spark for an idea about what could happen next>
  2. <a *DIFFERENT* idea for what could happen next>
  3. <another SHORT alternative idea for what could happen next>
  $output = [this.joinItems("\n")]

commentsOptions
  width = min(750px, 100%)
  height = min(70vh, 600px)
  bannedUsers
    3985b688818bb08c93c5
    
$meta 
  title = AI Story Generator (free, unlimited, no sign-up)
  description = Completely free & unlimited AI story generator/writer based on a prompt. No sign-up or login. Generate LONG stories, paragraph-by-paragraph, optionally guiding the AI on what happens next. Fast generation and there are no daily usage restrictions - unlimited and 100% free. You can prompt the AI to create horror stories (including creepy/creepypasta and analogue horror stories), funny stories, fantasy, mystery, anime, and basically anything else. Can do short stories, long stories - you could even try using this AI to write a novel! An AI storyteller.
  
  
resetRatingButtons() =>
  rateLastMessageBadBtn.disabled = true;
  rateLastMessageGoodBtn.disabled = true;
  rateLastMessageBadBtn.style.opacity = 1;
  rateLastMessageGoodBtn.style.opacity = 1;
  
enableRatingButtons() =>
  rateLastMessageBadBtn.disabled = false;
  rateLastMessageGoodBtn.disabled = false;
  
async rateLastMessage(rating) =>
  if(!window.lastGenerationStreamObj) return;
  
  if(!localStorage.knowsHowRatingsWork) {
    if(!confirm("Your ratings help improve Perchance's AI plugin, which powers this generator. Please do not submit ratings if your story includes sensitive personal info.\n\nContinue?")) return;
    localStorage.knowsHowRatingsWork = "1";
  }
  
  let score = rating==="good" ? 1 : 0;
  rateLastMessageBadBtn.disabled = true;
  rateLastMessageGoodBtn.disabled = true;
  if(rating === "good") {
    rateLastMessageBadBtn.style.opacity = 0.2;
  } else {
    rateLastMessageGoodBtn.style.opacity = 0.2;
  }
  
  if(!window.recentRatingReasonCounts) window.recentRatingReasonCounts = {};
  let reasonCountEntries = Object.entries(window.recentRatingReasonCounts).sort((a,b) => b[1]-a[1]);
  if(reasonCountEntries.length > 10) reasonCountEntries = reasonCountEntries.slice(0, 10);
  window.recentRatingReasonCounts = Object.fromEntries(reasonCountEntries);
  recentRatingReasonsDataList.innerHTML =  reasonCountEntries.map(e => `<option value="${e[0].replace(/</g, "&lt;").replace(/"/g, "&quot;")}"></option>`).join("");
  
  let reasonResolver;
  let reasonFinishPromise = new Promise(r => reasonResolver=r);
  ratingReasonEl.value = "";
  ratingReasonCtn.style.display = "";
  ratingReasonEl.focus();
  await new Promise(r => setTimeout(r, 100));
  
  // if they click anywhere other than the reason input, then we resolve with the current contents of the reason box
  function windowClickHandler(event) {
    if(!ratingReasonCtn.contains(event.target)) {
      reasonResolver(ratingReasonEl.value);
    }
  }
  window.addEventListener("click", windowClickHandler);
  
  // if they press enter, then we resolve too
  function enterKeydownHandler(event) {
    if(event.key === 'Enter') {
      reasonResolver(ratingReasonEl.value);
    }
  }
  ratingReasonEl.addEventListener("keydown", enterKeydownHandler);
  
  let reason = await reasonFinishPromise;
  if(reason.length < 100) window.recentRatingReasonCounts[reason] = (window.recentRatingReasonCounts[reason] || 0) + 1;
  
  ratingReasonCtn.style.display = 'none';
  window.removeEventListener("click", windowClickHandler);
  ratingReasonEl.removeEventListener("keydown", enterKeydownHandler);
  window.lastGenerationStreamObj.submitUserRating({score, reason});


bannedUsers // for comments section
  89f207af4524732bc398
  81cc368aeeb66ffda8ca
