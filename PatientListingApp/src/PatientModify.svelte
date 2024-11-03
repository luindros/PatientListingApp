<script lang="ts">
  import { onMount } from 'svelte';
  import { navigate } from "svelte-routing";
  import axios from "axios";

  export let id: string;

  const hapiFhir = "https://hapi.fhir.org/baseR4";
  let patient: any = null;
 // <td>{patient.name?.[0]?.family}, {patient.name?.[0]?.given?.join(' ') || ''}</td>
  onMount(async () => {
    try {
      const response = await axios.get(`${hapiFhir}/Patient/${id}`);
      patient = response.data;       

      const name = patient.name?.[0]
      firstName = name?.given?.[0]
      lastName = name?.family
      gender = patient.gender
      birthDate = patient.birthDate
      phoneNumber = patient.telecom?.[0]?.value

    } catch (error) {
      console.error("Error fetching patient:", error);
      error = true;
      
      setTimeout(() => {
        window.location.href = '/';  // Use window.location.href for a hard redirect
      }, 100); 
    }
  });

  let firstName: string | undefined;
  let lastName: string | undefined
  let gender: string | undefined
  let birthDate: string | undefined
  let phoneNumber: string | undefined

  async function updatePatient() {
    try {
      console.info("Hi")
      // Update the patient object with the form values
      patient.name[0].family = lastName;
      console.info("Family",patient.name[0].family)
      patient.name[0].given[0] = firstName;
      console.info("Given",patient.name[0].given[0])
      patient.gender = gender;
      console.info("Gender",patient.gender)
      patient.birthDate = birthDate;
      console.info("BirthDate",patient.birthDate)
      patient.telecom[0].value = phoneNumber;
      console.info("PhoneNumber",patient.telecom[0].value)


      console.info("Patient",patient)

      await axios.put(`${hapiFhir}/Patient/${id}`, patient);
      alert("Patient updated successfully!");
      navigate("/");
    } catch (error) {
      console.error("Error updating patient:", error);
      alert("Failed to update patient. Please try again.");
    }
  }

  async function deletePatient() {
  if (confirm("Are you sure you want to delete this patient?")) {
    try {
      await axios.delete(`${hapiFhir}/Patient/${id}`);
      alert("Patient deleted successfully!");
      navigate("/");
    } catch (error) {
      console.error("Error deleting patient:", error);
      alert("Failed to delete patient. Please try again.");
    }
  }
}
</script>

<main>
  <div class="search-container">
    <h1 class="title">Modify Patient</h1>
    {#if patient}
      <form on:submit|preventDefault={updatePatient}>
        <label>
          Family Name:
          <input bind:value={lastName} required>
        </label>
        <label>
          Given Name:
          <input bind:value={firstName} required>
        </label>
        <label>
          Gender:
          <select bind:value={gender} required>
           <option value="" disabled>Select gender</option>
            <option value="male">Male</option>
            <option value="female">Female</option>
            <option value="other">Other</option>
            <option value="unknown">Unknown</option>
          </select>
        </label>
        <label>
          Birth Date:
          <input type="date" bind:value={birthDate} required>
        </label>
        <label>
          Phone Number:
          <input type="tel" bind:value={phoneNumber}>
        </label>
        <div class="button-container">
        <button type="submit">Update Patient</button>
        <button type="button" on:click={deletePatient}>Delete Patient</button>
      </form>
    {:else}
      <p>Loading patient data...</p>
    {/if}
  </main>
  
  <style>
    .title{
      text-align: center;
      color: rgb(35, 166, 184);
    }
    form {
      display: flex;
      flex-direction: column;
      gap: 10px;
      max-width: 300px;
      margin: 0 auto;
    }
    label {
      display: flex;
      flex-direction: column;
    }
    input, select {
      margin-top: 5px;
      padding: 5px;
    }
    button {
      padding: 10px 20px;
      font-size: 16px;
      background-color: #48a2ad;
      color: white;
      border: none;
      border-radius: 0 4px 4px 0;
      cursor: pointer;
      transition: background-color 0.3s ease;
    }
    button:hover {
      top: 10px;
      padding: 10px 20px;
      background-color:  #82edfd;
    }
   
  </style>