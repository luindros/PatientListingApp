<script lang="ts">
    import { onMount } from 'svelte';
    import axios from "axios";
    import { navigate } from "svelte-routing";

    export let id: string;
    const hapiFhir = "https://hapi.fhir.org/baseR4";
    let questionnaires: any[] = [];
    let loading = true;
    let error = false;

    onMount(async () => {
        await fetchQuestionnaires();
    });

    async function fetchQuestionnaires() {
        loading = true;
        error = false;
        try {
            const response = await axios.get(`${hapiFhir}/QuestionnaireResponse`, {
                params: {
                    subject: `Patient/${id}`,
                    _sort: '-authored',
                    _count: 100
                }
            });
            questionnaires = response.data.entry?.map((entry: any) => entry.resource) || [];
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
            {#if questionnaires.length === 0}
                <p>No questionnaires found for this patient.</p>
            {:else}
                <table>
                    <thead>
                        <tr>
                            <th>Date</th>
                            <th>Questionnaire ID</th>
                            <th>Questionnaire</th>
                            <th>Responses</th>
                            <th>Encounter ID</th>
                            <th>Actions</th>
                        </tr>
                    </thead>
                    <tbody>
                        {#each questionnaires as questionnaire}
                            <tr>
                                <td>{new Date(questionnaire.authored).toLocaleString()}</td>
                                <td>{questionnaire.id || 'N/A'}</td>
                                <td>
                                    {#each questionnaire.item as item}
                                        <div>{item.text || 'Unknown'}</div>
                                    {/each}
                                </td>
                                <td>
                                    {#each questionnaire.item as item}
                                        <div>
                                            {#each item.answer as answer}
                                                <div>{getAnswerValue(answer)}</div> <!-- Display all answer values -->
                                            {/each}
                                        </div>
                                    {/each}
                                </td>
                                <td>{questionnaire.identifier?.value || 'N/A'}</td>
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
        background-color: #005f69;
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
</style>