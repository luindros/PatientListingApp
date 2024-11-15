<script lang="ts">
    import { onMount } from 'svelte';
    import axios from "axios";
    import { navigate } from "svelte-routing";

    export let id: string; // Patient ID
    const hapiFhir = "https://hapi.fhir.org/baseR4";
    let questionnaires: any[] = []; // Array to hold questionnaire responses
    let patient: any = null; // To store patient details
    let loading = true;
    let error = false;

    // Define types for FHIR resources
    interface Encounter {
        resourceType: string;
        id: string;
        identifier: Array<{ value: string }>; // Identifier array
        reference: string; // Reference to the encounter
    }

    interface QuestionnaireResponse {
        resourceType: string;
        id: string;
        authored: string;
        item: Array<{
            linkId: string;
            text: string;
            answer?: Array<{
                valueString?: string;
                valueBoolean?: boolean;
                valueInteger?: number;
                valueDecimal?: number;
                valueDate?: string;
            }>;
        }>;
        encounter?: {
            reference: string;
        };
    }

    interface Bundle {
        resourceType: string;
        entry: Array<{
            resource: QuestionnaireResponse | Encounter; // Can be either QuestionnaireResponse or Encounter
        }>;
    }

    onMount(async () => {
        await fetchPatientDetails(); // Fetch patient details on component mount
        await fetchQuestionnaires(); // Fetch questionnaires
    });

    async function fetchPatientDetails() {
        try {
            const response = await axios.get(`${hapiFhir}/Patient/${id}`);
            patient = response.data; // Store patient details
        } catch (err) {
            console.error("Error fetching patient details:", err);
            error = true;
        }
    }

    async function fetchQuestionnaires() {
        loading = true;
        error = false;
        try {
            const response = await axios.get<Bundle>(`${hapiFhir}/QuestionnaireResponse`, {
                params: {
                    subject: `Patient/${id}`,
                    _include: "QuestionnaireResponse:encounter", // Include encounter information
                    _sort: '-authored',
                    _count: 100
                }
            });

            // Extract questionnaire responses from the response
            questionnaires = response.data.entry
                .filter(entry => entry.resource.resourceType === 'QuestionnaireResponse')
                .map(entry => {
                    const questionnaire = entry.resource;

                    // Find the encounter resource associated with this questionnaire
                    const encounterEntry = response.data.entry.find(e => e.resource.resourceType === 'Encounter' && e.resource.id === questionnaire.encounter?.reference.split('/')[1]);
                    const encounterIdentifier = encounterEntry?.resource.identifier?.[0]?.value || 'N/A'; // Get the encounter identifier

                    return {
                        ...questionnaire,
                        encounterIdentifier // Add the encounter identifier to the questionnaire object
                    };
                });
        } catch (err) {
            console.error("Error fetching questionnaires:", err);
            error = true;
        } finally {
            loading = false;
        }
    }

    function getAnswerValue(answer: any): string {
        if (answer.valueString !== undefined) {
            return answer.valueString;
        } else if (answer.valueBoolean !== undefined) {
            return answer.valueBoolean ? 'Yes' : 'No';
        } else if (answer.valueInteger !== undefined) {
            return answer.valueInteger.toString();
        } else if (answer.valueDecimal !== undefined) {
            return answer.valueDecimal.toString();
        } else if (answer.valueDate !== undefined) {
            return new Date(answer.valueDate).toLocaleDateString();
        } else {
            return 'N/A';
        }
    }

    async function deleteQuestionnaire(questionnaireId: string) {
        try {
            await axios.delete(`${hapiFhir}/QuestionnaireResponse/${questionnaireId}`);
            await fetchQuestionnaires(); // Refresh the list after deletion
        } catch (err) {
            console.error("Error deleting questionnaire:", err);
            alert("Failed to delete questionnaire. Please try again.");
        }
    }
</script>

<main>
    <div class="container">
        <h1 class="title">Patient Questionnaires</h1>
        {#if loading}
            <p>Loading questionnaires...</p>
        {:else if error}
            <p class="error">Error loading questionnaires. Please try again.</p>
        {:else}
            {#if patient}
                <h2 class="patientName">{patient.name[0].given.join(' ')} {patient.name[0].family} ({patient.id})</h2>
            {/if}
            
            {#if questionnaires.length === 0}
                <p>No questionnaires found for this patient.</p>
            {:else}
                <table>
                    <thead>
                        <tr>
                            <th>Date</th>
                            <th>Questionnaire ID</th>
                            <th>Questions and Responses</th>
                            <th>Encounter ID</th>
                            <th>Encounter Identifier</th> <!-- New column for Encounter Identifier -->
                            <th>Actions</th>
                        </tr>
                    </thead>
                    <tbody>
                        {#each questionnaires as questionnaire}
                            <tr>
                                <td>{new Date(questionnaire.authored).toLocaleString()}</td>
                                <td>{questionnaire.id || 'N/A'}</td>
                                <td>
                                    <div class="qa-section">
                                        {#each questionnaire.item as item}
                                            <div class="question-response-pair">
                                                <div class="question">{item.text || 'Unknown'}</div>
                                                <div class="response">
                                                    {#if item.answer}
                                                        {#each item.answer as answer}
                                                            {getAnswerValue(answer)} 
                                                        {/each}
                                                    {:else}
                                                        No response
                                                    {/if}
                                                </div>
                                            </div>
                                        {/each}
                                    </div>
                                </td>
                                <td>{questionnaire.encounter?.reference || 'N/A'}</td> <!-- Display Encounter ID -->
                                <td>{questionnaire.encounterIdentifier || 'N/A'}</td> <!-- Display Encounter Identifier -->
                                <td>
                                    <button class="delete-button" on:click={() => deleteQuestionnaire(questionnaire.id)}>
                                        Delete
                                    </button>
                                </td>
                            </tr>
                        {/each}
                    </tbody>
                </table>
            {/if}
            <div class="button-container">
                <button on:click={() => navigate(-1)}>Back</button>
            </div>
        {/if}
    </div>
</main>

<style>
    .container {
        max-width: 1200px;
        margin: 0 auto;
        padding: 20px;
    }

    .title {
        text-align: center;
        color: rgb(35, 166, 184);
    }

    .patientName {
        text-align: center;
        color: rgb(0, 0, 0);
    }

    table {
        width: 100%;
        border-collapse: separate;
        border-spacing: 0;
        margin-top: 20px;
        box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
        border-radius: 8px;
        overflow: hidden;
    }

    th, td {
        padding: 12px;
        text-align: left;
        border-bottom: 1px solid #e0e0e0;
    }

    th {
        background-color: #00adc0;
        color: white;
        font-weight: bold;
        text-transform: uppercase;
        letter-spacing: 0.5px;
    }

    tr:last-child td {
        border-bottom: none;
    }

    tbody tr:hover {
        background-color: #a8b4b6;
        transition: background-color 0.3s ease;
    }

    .error {
        color: red;
        font-weight: bold;
    }

    .button-container {
        display: flex;
        justify-content: center;
        margin-top: 20px;
    }

    button {
        padding: 10px 20px;
        font-size: 16px;
        background-color: #48a2ad;
        color: white;
        border: none;
        border-radius: 4px;
        cursor: pointer;
        transition: background-color 0.3s ease;
        margin: 0 5px;
    }

    button:hover {
        background-color: #82edfd;
    }

    .delete-button {
        background-color: #f44336;
        padding: 5px 10px;
        font-size: 14px;
    }

    .delete-button:hover {
        background-color: #d32f2f;
    }

    /* New styles for question-response pairs */
    .qa-section {
        margin-top: 10px;
    }

    .question-response-pair {
        margin-bottom: 8px;
        padding: 8px;
        border: 1px solid #7aa9b0; /* Light blue border */
        border-radius: 5px;
        background-color: #e0f7fa; /* Light blue background for questions */
    }

    .question {
        font-weight: bold;
        color: #00796b; /* Darker color for questions */
    }

    .response {
        margin-left: 10px; /* Indent responses for clarity */
        color: #d32f2f; /* Darker color for responses */
    }
</style>