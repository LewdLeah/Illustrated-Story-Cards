To use this little tool, export your story cards file, fill in the 2 commented inputs at the bottom (story cards array and OpenRouter API key), run the script externally (I used VS Code), and then find the newly created images folder:
```javascript
require("dotenv").config();
const fs = require("fs");
const path = require("path");
const generateImages = async ({
    prompt,
    attempts = 1,
    filename,
    aspectRatio = "3:2",
    model = "google/gemini-2.5-flash-image",
    useConsole = false,
    outputPath = "./",
    folder = "images",
    title = "Illustrated Story Cards",
    apiKey
} = {}) => {
    if (!prompt || typeof prompt !== "string") {
        throw new Error("A valid \"prompt\" string is required");
    }
    if (!apiKey) {
        throw new Error("Missing API key for OpenRouter");
    }
    const folderPath = `${outputPath}${folder}`;
    if (!fs.existsSync(folderPath)) {
        fs.mkdirSync(folderPath, { recursive: true });
    }
    const request = {
        method: "POST",
        headers: {
            "Authorization": `Bearer ${apiKey}`,
            "Content-Type": "application/json",
            "HTTP-Referer": "https://LewdLeah.app",
            "X-Title": title
        },
        body: JSON.stringify({
            model,
            messages: [{
                role: "user",
                content: prompt.trim()
            }],
            modalities: ["image", "text"],
            image_config: { aspect_ratio: aspectRatio }
        })
    };
    if (!filename) {
        filename = new Date().toISOString().replace(/[:.]/g, "-");
    }
    let images;
    for (let attempt = 1; attempt <= attempts; attempt++) {
        try {
            const response = await fetch("https://openrouter.ai/api/v1/chat/completions", request);
            if (!response.ok) {
                throw new Error(`HTTP ${response.status} ${response.statusText} - ${await response.text()}`);
            }
            images = (await response.json())?.choices?.[0]?.message?.images;
            if (images && Array.isArray(images) && (0 < images.length)) {
                break;
            } else {
                throw new Error("No images returned by model");
            }
        } catch (error) {
            console.error(`Attempt #${attempt} for "${filename}" failed: ${error.message}`);
            if (attempt === attempts) {
                throw error;
            } else {
                await new Promise(resolve => setTimeout(resolve, 30000));
            }
        }
    }
    const savedPaths = [];
    for (let i = 0; i < images.length; i++) {
        const url = images[i]?.image_url?.url;
        if (!url) {
            continue;
        }
        try {
            const imgRes = await fetch(url);
            if (!imgRes.ok) {
                throw new Error(`Failed to fetch image: ${url}`);
            }
            const buffer = Buffer.from(await imgRes.arrayBuffer());
            const filePath = path.join(folderPath, `${filename.replace(/\.png$/, "")}${(1 < images.length) ? `-${i + 1}` : ""}.png`);
            fs.writeFileSync(filePath, buffer);
            savedPaths.push(filePath);
            if (useConsole) {
                console.log(`Saved image ${i + 1}: ${filePath}`);
            }
        } catch (error) {
            console.error(`Error saving image ${i + 1}: ${error.message}`);
        }
    }
    return savedPaths;
};
const pictureBook = async ({ storyCards = [], apiKey = "", lore = "" } = {}) => {
    lore = lore.trim();
    for (const card of storyCards) {
        const norm = (s) => (s
            .replace(/[#*_~•·∙⋅]+/g, "")
            .normalize("NFKD").replace(/[\u0300-\u036f]/g, "")
            .replace(/\u00A0|\u2000-\u200B|\u202F|\u205F|\u3000/g, " ")
            .trim()
            .replace(/\s+/g, " ")
            .replaceAll("…", "...")
            .replace(/[“”«»„‟]/g, "\"")
            .replace(/[‘’‚‛]/g, "'")
            .replaceAll("s's", "s'")
            .replace(/[–−‒]/g, "-")
            .replaceAll(" - ", "—")
            .replace(/ *— */g, "—")
        );
        generateImages({
            folder: "images",
            filename: `${norm(card.title).replace(/[^a-z0-9 _-]/gi, "").trim().replace(/^The +/i, "").replace(/\s+/g, "_").replace(/_+/, "_")}.png`,
            prompt: `${lore ? `Broader context:\n${lore}\n\n` : ""}# Create an anime-style illustration (no text) for the following passage:\n${norm(card.value)}`,
            useConsole: true,
            attempts: 6,
            apiKey
        }).catch(error => {
            console.error(error.message);
        });
        const randInt = (min, max) => Math.floor(Math.random() * (max - min + 1)) + min;
        await new Promise(resolve => setTimeout(resolve, randInt(2000, 4000)));
    }
    console.log("All done!");
    return;
};
(async () => {
    await pictureBook({
        storyCards: [], // Put your exported story cards array here
        apiKey: "", // OpenRouter API key to generate your story card images
        lore: "" // Optional: Include some extra info/context about your scenario
    });
    return;
})();
```
I hope you will enjoy~ ❤️
