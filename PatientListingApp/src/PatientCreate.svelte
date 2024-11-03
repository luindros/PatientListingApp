<script lang="ts">
    import { navigate } from "svelte-routing";
    import axios from "axios";
    
  
    const hapiFhir = "https://hapi.fhir.org/baseR4";
  
    let familyName = "";
    let givenName = "";
    let gender = "";
    let birthDate = "";
    let phoneNumber = ""; // New field for phone number
  
    async function createPatient() {
      const patient = {
        resourceType: "Patient",
        name: [{ family: familyName, given: [givenName] }],
        gender: gender,
        birthDate: birthDate,
        telecom: [{ system: "phone", value: phoneNumber }] // Include phone number
      };
  
      try {
        await axios.post(`${hapiFhir}/Patient`, patient);
        navigate("/");
      } catch (error) {
        console.error("Error creating patient:", error);
      }
    }
  </script>
  
  <main>
    <div class="search-container">
      <h1 class="title">Create Patient</h1>
      <form on:submit|preventDefault={createPatient}>
        <label>
          Family Name:
          <input bind:value={familyName} required>
        </label>
        <label>
          Given Name:
          <input bind:value={givenName} required>
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
        <button type="submit">Create Patient</button>
      </form>
    </div>
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
      background-color: #82edfd;
    }
  </style>