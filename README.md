
/**
 * @license
 * SPDX-License-Identifier: Apache-2.0
 */
import { GoogleGenAI } from '@google/genai';
import { marked } from 'marked';
import DOMPurify from 'dompurify';

// --- Global state ---
let ai: GoogleGenAI;
let chat: any;

/**
 * Converts a File object to a base64 encoded string.
 * @param file The file to convert.
 * @returns A promise that resolves with the base64 string.
 */
function fileToBase64(file: File): Promise<string> {
    return new Promise((resolve, reject) => {
        const reader = new FileReader();
        reader.readAsDataURL(file);
        reader.onload = () => resolve(reader.result as string);
        reader.onerror = error => reject(error);
    });
}

// --- Main App Initialization (runs after DOM is loaded) ---

async function main() {
    // --- DOM Elements ---
    const chatHistory = document.getElementById('chat-history');
    const chatForm = document.getElementById('chat-form');
    const chatInput = document.getElementById('chat-input') as HTMLInputElement;
    const sendButton = document.getElementById('send-button') as HTMLButtonElement;
    const bookingForm = document.getElementById('booking-form') as HTMLFormElement;
    const bookingButton = bookingForm?.querySelector('button') as HTMLButtonElement;

    // New form elements
    const petPhotoInput = document.getElementById('pet-photo-input') as HTMLInputElement;
    const petPhotoPreview = document.getElementById('pet-photo-preview') as HTMLImageElement;
    const firstTimeSelect = document.getElementById('first-time') as HTMLSelectElement;
    const previousExperienceGroup = document.getElementById('previous-experience-group');
    const checkInInput = document.getElementById('check-in') as HTMLInputElement;
    const checkOutInput = document.getElementById('check-out') as HTMLInputElement;

    // Chat action buttons
    const askPlaypenBtn = document.getElementById('ask-playpen-btn');

    // Modal elements
    const viewRatesBtn = document.getElementById('view-rates-btn');
    const ratesModal = document.getElementById('rates-modal');
    const closeModalBtn = document.getElementById('close-modal-btn');


    if (!chatHistory || !chatForm || !bookingForm) {
        console.error("Critical elements not found. App cannot start.");
        document.body.innerHTML = `<div style="text-align: center; padding: 2rem; font-family: sans-serif; color: #d32f2f;">
            <h2>Application Error</h2>
            <p>Could not load essential page components. Please try refreshing.</p>
        </div>`;
        return;
    }

    // --- Core Functions (that interact with the DOM) ---

    function createMessageElement(sender: 'user' | 'ai' | 'error' | 'loading') {
        if (!chatHistory) return null;
        const messageElement = document.createElement('div');
        messageElement.classList.add('chat-message', `${sender}-message`);
        chatHistory.appendChild(messageElement);
        chatHistory.scrollTop = chatHistory.scrollHeight;
        return messageElement;
    }

    function addMessage(text: string, sender: 'user' | 'ai' | 'error') {
        const messageElement = createMessageElement(sender);
        if (messageElement) {
            if (sender === 'ai') {
                // Render AI messages as sanitized HTML to support markdown
                messageElement.innerHTML = DOMPurify.sanitize(marked(text) as string);
            } else if (sender === 'user' && text.startsWith('<img')) {
                 // This handles a potential edge case for user messages containing HTML
                messageElement.innerHTML = text;
            } else {
                // Render user and error messages as plain text
                messageElement.textContent = text;
            }
        }
    }

    function toggleLoading(isLoading: boolean) {
        if (chatInput) chatInput.disabled = isLoading;
        if (sendButton) sendButton.disabled = isLoading;
        if (bookingButton) bookingButton.disabled = isLoading;

        const existingSpinner = chatHistory?.querySelector('.loading-message');
        if (isLoading && !existingSpinner) {
            const spinner = createMessageElement('loading');
            if (spinner) spinner.innerHTML = '<div class="dot-flashing"></div>';
        } else if (!isLoading && existingSpinner) {
            existingSpinner.remove();
        }
    }

    async function handleAiResponse(prompt: any) {
        toggleLoading(true);

        try {
            const responseStream = await chat.sendMessageStream({ message: prompt });

            let currentAiMessageElement = createMessageElement('ai');
            if (!currentAiMessageElement) {
                addMessage('Could not display the AI response.', 'error');
                toggleLoading(false);
                return;
            }

            let aiResponseText = '';
            for await (const chunk of responseStream) {
                // Ensure chunk and chunk.text are valid before processing
                if (chunk && typeof chunk.text === 'string') {
                    aiResponseText += chunk.text;
                    // Render the markdown response as sanitized HTML, updating on each chunk
                    currentAiMessageElement.innerHTML = DOMPurify.sanitize(marked(aiResponseText) as string);
                    if (chatHistory) chatHistory.scrollTop = chatHistory.scrollHeight;
                }
            }
        } catch (error) {
            console.error('Error sending message to Gemini:', error);
            addMessage('Oops! Something went wrong. Please try asking again.', 'error');
        } finally {
            toggleLoading(false);
        }
    }

    async function handleUserMessage(message: string) {
        addMessage(message, 'user');
        if (chatInput) chatInput.value = '';
        if (!chat) {
            addMessage('The chat is not available right now. Please try again later.', 'error');
            return;
        }
        await handleAiResponse(message);
    }

    // --- Event Listeners ---
    chatForm?.addEventListener('submit', async (e) => {
        e.preventDefault();
        if (!chatInput) return;
        const message = chatInput.value.trim();
        if (message) {
            await handleUserMessage(message);
        }
    });

    bookingForm?.addEventListener('submit', async (e) => {
        e.preventDefault();
        if (!bookingForm) return;

        if (!bookingForm.checkValidity()) {
            bookingForm.reportValidity();
            addMessage('Please fill out all required fields before booking.', 'error');
            return;
        }

        if (!checkInInput || !checkOutInput || !checkInInput.value || !checkOutInput.value || new Date(checkInInput.value) >= new Date(checkOutInput.value)) {
            addMessage('Check-out date must be after the check-in date, and both must be filled out.', 'error');
            return;
        }

        const formData = new FormData(bookingForm);
        const rabbitName = formData.get('rabbit-name') as string;
        const imageFile = petPhotoInput?.files?.[0];
        const promptParts: any[] = [];

        let textPrompt = `A new booking request has been submitted. Please confirm the details.\n\n**Rabbit's Profile:**\n`;
        const fieldLabels: { [key: string]: string } = {
            'rabbit-name': "Name", 'gender': "Gender", 'age': "Age", 'breed': "Breed",
            'medical-condition': "Medical Conditions", 'temperament': "Temperament",
            'favorites': "Favorite Foods", 'habits': "Habits/Personality", 'routine': "Daily Routine",
            'special-requirements': "Special Requirements", 'sterilised': "Sterilised", 'vaccinated': "Vaccinated",
            'first-time': "First Time Boarding", 'previous-experience': "Previous Experience",
            'check-in': "Check-in", 'check-out': "Check-out"
        };

        for (const [key, value] of formData.entries()) {
            if (key === 'pet-photo' || !value) continue;
            if (key === 'previous-experience' && firstTimeSelect?.value === 'Yes') continue;

            const label = fieldLabels[key] || key;
            textPrompt += `- **${label}**: ${value}\n`;
        }

        if (imageFile && petPhotoInput) {
            try {
                const base64Image = await fileToBase64(imageFile);
                promptParts.push({
                    inlineData: { mimeType: imageFile.type, data: base64Image.split(',')[1] }
                });
                addMessage(`Sending booking request for ${rabbitName} with a photo...`, 'user');
            } catch (error) {
                console.error("Image processing error:", error);
                addMessage("Sorry, there was an error processing the photo.", 'error');
                return;
            }
        } else {
            addMessage(`Sending booking request for ${rabbitName}...`, 'user');
        }

        promptParts.push({ text: textPrompt });
        
        document.getElementById('chat-section')?.scrollIntoView({ behavior: 'smooth' });
        await handleAiResponse(promptParts);
        bookingForm.reset();
        if (petPhotoPreview) {
            petPhotoPreview.src = '';
            petPhotoPreview.classList.add('hidden');
        }
    });

    askPlaypenBtn?.addEventListener('click', async () => {
        const prompt = "Tell me about your playpen setup, and please include a picture.";
        await handleUserMessage(prompt);
    });

    petPhotoInput?.addEventListener('change', () => {
        if (!petPhotoInput || !petPhotoPreview) return;
        const file = petPhotoInput.files?.[0];
        if (file) {
            const reader = new FileReader();
            reader.onload = (e) => {
                if (e.target?.result) {
                    petPhotoPreview.src = e.target.result as string;
                    petPhotoPreview.classList.remove('hidden');
                }
            }
            reader.readAsDataURL(file);
        }
    });

    firstTimeSelect?.addEventListener('change', () => {
        if (!firstTimeSelect || !previousExperienceGroup) return;
        if (firstTimeSelect.value === 'No') {
            previousExperienceGroup.classList.remove('hidden');
        } else {
            previousExperienceGroup.classList.add('hidden');
        }
    });

    // Modal listeners
    viewRatesBtn?.addEventListener('click', (e) => {
        e.preventDefault();
        ratesModal?.classList.add('visible');
    });

    closeModalBtn?.addEventListener('click', () => {
        ratesModal?.classList.remove('visible');
    });

    ratesModal?.addEventListener('click', (e) => {
        // Close modal if user clicks on the overlay (the background)
        if (e.target === ratesModal) {
            ratesModal.classList.remove('visible');
        }
    });

    // --- AI Initialization ---
    try {
        ai = new GoogleGenAI({ apiKey: process.env.API_KEY });
        chat = ai.chats.create({
            model: 'gemini-2.5-flash',
            config: {
                systemInstruction: `You are a friendly and helpful virtual assistant for "Bunny BNB", a premium rabbit boarding service. Your goal is to answer user questions and assist with booking requests.

      Key information about Bunny BNB:
      - **Services**: Your bunny's stay includes a designated cage, all-day free roam, comfortable flooring, daily cleaning of litter box and water, air circulation via a fan, and 24-hour CCTV updates.
      - **Pricing**: Boarding rates depend on the length of stay.
        - 1 to 5 nights: $35 per night.
        - 6 or more nights: $20 per night (special rate).
      - **Check-in/Check-out**: All check-ins and check-outs are strictly by appointment only. Please provide your estimated arrival/departure time 24 hours in advance. A late fee applies for arrivals or departures after 10:00 PM.
      - **Food & Water**: We provide unlimited CHEWBO (2nd cut) hay, pellets, and filtered water. Fresh vegetables and fruits are served on alternate days. Owners are welcome to bring their own food to avoid digestive issues.
      - **Playpen Setup**: Our playpens are spacious (4x4 feet) with soft flooring. Each includes a large litter box with rabbit-safe litter, a hay feeder, a water bowl, and enrichment toys. You can see a sample setup here: [View a photo of our playpen](https://photos.app.goo.gl/26nLPfSHMzf4kjuH8)
      - **Booking**: When a user submits a booking form, all their rabbit's details will be provided. Your job is to acknowledge the detailed request, confirm the key details (rabbit's name, check-in, check-out), express excitement about meeting the rabbit, and state that a team member will email them shortly to finalize everything. Do not ask for the information again. Be friendly and enthusiastic.
      - **Photo Analysis**: When a user sends a photo of their pet, assess its general size and appearance. Reassure the user that your playpen setup is safe and comfortable for a rabbit like theirs. Mention key features of the playpen (e.g., "A rabbit of that size would have plenty of room in our 4x4 foot playpens, and the soft flooring is great for sensitive paws."). Be positive and welcoming.

      Always be polite and use a warm, caring tone. Use markdown for formatting, like lists or bold text, to make your answers clear and easy to read.`,
            },
        });
        // Send initial welcome message
        addMessage('Hello! How can I help you with your rabbit boarding needs today?', 'ai');
    } catch (error) {
        console.error('Failed to initialize GoogleGenAI:', error);
        addMessage('Sorry, I am unable to connect to the AI service right now. Please try again later.', 'error');
        if (chatInput) chatInput.disabled = true;
        if (sendButton) sendButton.disabled = true;
        if (bookingButton) bookingButton.disabled = true;
    }
}


// --- App Entry Point ---
if (document.readyState === 'loading') {
    document.addEventListener('DOMContentLoaded', main);
} else {
    // DOMContentLoaded has already fired
    main();
}
