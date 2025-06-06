// === CONFIGURATION === 
const AIRTABLE_PAT = 'pat159vsLfEOmkB28.1dc0ffd528611c9e4e1576280d45f66741f58b467d9892f48f77bb5b3ab3565d'; 
const AIRTABLE_BASE_ID = 'appCDZB437amxaRPq'; 
const AIRTABLE_TABLE_NAME = 'Clients';
const OPENAI_API_KEY = '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'; 
// your OpenAI API key const MODEL = 'gpt-3.5-turbo'; const COST_PER_1K_TOKENS = 0.002; // $0.002 per 1K tokens const BUDGET_DOLLARS = 10.00; // your total budget cap

// Entry function to run the whole flow
function runGeneratePlansFromAirtable() {
  try {
    const participants = fetchParticipantsFromAirtable();
    Logger.log(`Fetched ${participants.length} participants from Airtable.`);
    generateImplementationPlans(participants);
  } catch (e) {
    Logger.log('Error in main flow: ' + e.message);
  }
}

// Fetch participants from Airtable API
function fetchParticipantsFromAirtable() {
  const url = `https://api.airtable.com/v0/${AIRTABLE_BASE_ID}/${encodeURIComponent(AIRTABLE_TABLE_NAME)}?pageSize=100`;
  
  const options = {
    method: 'get',
    headers: { Authorization: `Bearer ${AIRTABLE_PAT}` },
    muteHttpExceptions: true
  };
  
  const response = UrlFetchApp.fetch(url, options);
  if (response.getResponseCode() !== 200) {
    throw new Error(`Airtable API error ${response.getResponseCode()}: ${response.getContentText()}`);
  }
  
  const data = JSON.parse(response.getContentText());
  
  return data.records.map(record => {
    const f = record.fields;
    return {
      name: f['Name'] || 'Unknown',
      legalStatus: f['Legal Status'] || 'Not Provided',
      languageSpoken: f['Language Spoken'] || 'Not Provided',
      primaryDiagnosis: f['Primary Diagnosis'] || 'Not Provided',
      dateOfBirth: f['Date of Birth'] || 'Not Provided',
      gender: f['Gender'] || 'Not Provided',
      race: f['Race'] || 'Not Provided',
      socialSecurity: f['Social Security Number'] || '⚠️',
      currentAddress: f['Current Address'] || '⚠️',
      phoneNumber: f['Phone Number'] || '⚠️',
      supportCoordinator: f['Support Coordinator'] || '⚠️',
      admissionDate: f['Admission Date'] || '⚠️',
      supportPlanReceived: f['Support Plan Received'] || '⚠️',
      supportPlanEffectiveDate: f['Support Plan Effective Date'] || '⚠️',
      implementationDate: f['Implementation Date'] || '⚠️',
      implementationEffectiveDate: f['Implementation Effective Date'] || '⚠️',
      // add any other fields you need here...
    };
  });
}

// Generate implementation plans for each participant
function generateImplementationPlans(participants) {
  participants.forEach(participant => {
    try {
      const prompt = buildFullPrompt(participant);
      if (!prompt || prompt.trim() === '') {
        Logger.log(`Empty prompt for ${participant.name}, skipping.`);
        return;
      }
      
      const openAIResponse = callOpenAI(prompt);
      const planText = extractOpenAIText(openAIResponse);
      
      if (!planText || planText.trim() === '') {
        Logger.log(`Empty OpenAI response for ${participant.name}, skipping doc creation.`);
        return;
      }
      
      createGoogleDoc(participant.name, planText);
      Logger.log(`Created doc for ${participant.name}`);
      
      Utilities.sleep(1000); // brief pause to avoid rate limits
      
    } catch (err) {
      Logger.log(`Error processing ${participant.name}: ${err.message}`);
    }
  });
}

// Build the detailed prompt including sample Implementation Plan + Annual Summary + participant data
function buildFullPrompt(p) {
  // Template incorporating your sample texts, inserting participant data dynamically
  return `
Implementation Plan

PARTIAL IMPLEMENTATION PLAN

**Participant Name:** ${p.name}

**Legal Status:** ${p.legalStatus}

**Language Spoken:** ${p.languageSpoken}

**Primary Diagnosis:** ${p.primaryDiagnosis}

**Date of Birth:** ${p.dateOfBirth}

**Gender:** ${p.gender}

**Race:** ${p.race}

**Social Security Number (last four digits):** ${p.socialSecurity}

**Current Address:** ${p.currentAddress}

**Phone Number:** ${p.phoneNumber}

**Support Coordinator:** ${p.supportCoordinator}

**Admission Date:** ${p.admissionDate}

**Support Plan Received:** ${p.supportPlanReceived}

**Support Plan Effective Date:** ${p.supportPlanEffectiveDate}

**Implementation Date:** ${p.implementationDate}

**Implementation Effective Date:** ${p.implementationEffectiveDate}

## State Quarters of IPP Year

## 1st: Nov/Dec/Jan

## 2nd: Feb/Mar/Apr

## 3rd: May/Jun/Jul

## 4th: Aug/Sept/Oct

IMPLEMENTATION PLAN SUMMARY

A meeting will be scheduled with ${p.name} and their support coordinator to discuss and determine the goals for their Implementation Plan.

INDIVIDUAL PROFILE:

${p.name} is an individual diagnosed with ${p.primaryDiagnosis}. Interests and goals will be discussed during the implementation planning meeting.

FUNCTIONAL STATUS:

Details about sensory functioning, daily living activities, communication, and behaviors will be assessed during the meeting to determine support needs.

PHYSICAL STATUS:

Physical health, medical visits, medications, and history of abuse or neglect will be reviewed.

COMMUNITY INCLUSION AND FULFILLMENT OF VALUED ADULT ROLES:

A plan will be developed to support active community participation and fulfillment of valued roles.

GOALS:

Goals will be chosen during the meeting, focused on improving independent living skills, and will be specific, measurable, achievable, relevant, and time-bound.

1. Goal #1: To improve independent living skills by [specific action].  
   - Objective 1: [Detailed step 1].  
   - Objective 2: [Detailed step 2].  
   - Objective 3: [Detailed step 3].

2. Goal #2: To enhance independent living skills by [specific action].  
   - Objective 1: [Detailed step 1].  
   - Objective 2: [Detailed step 2].  
   - Objective 3: [Detailed step 3].

METHODOLOGY STRATEGIES:

Staff will provide guidance, prompting, and assistance to support goal achievement. Training includes step-by-step instructions, demonstrations, and encouragement.

SYSTEM DATA COLLECTION/ASSESSMENT PROGRESS:

Daily documentation on a data sheet, monthly summaries, and quarterly reports will track progress and independence level.

MEDICAL VISITS:

Upcoming medical appointments will be scheduled and tracked. Medication or health status changes will be documented.

ABUSE, NEGLECT, AND EXPLOITATION:

Discussions will be held to educate about recognizing and reporting abuse. Hotline numbers will be provided.

–––––

ANNUAL SUMMARY 2024-2025

CONSUMER INFORMATION

- **Individual’s information:**  
  - **Name:** ${p.name}  
  - **Diagnosis:** ${p.primaryDiagnosis}  
  - **Interests:** [Insert interests if known]

- **DOB:** ${p.dateOfBirth}  
- **Medicaid number:** ⚠️  
- **Social security number:** ${p.socialSecurity}  
- **Recipient ID:** ⚠️  
- **PIN:** ⚠️  
- **Status:** ⚠️  
- **Date of Admission:** ${p.admissionDate}

- **Residence:** ${p.currentAddress}  
- **Phone number:** ${p.phoneNumber}

- **Next of kin:** ⚠️  
  - **Relationship:** ⚠️  
  - **Phone number:** ⚠️

- **Emergency contact:** ⚠️

- **SP Effective Date:** ⚠️  
- **Annual Summary Date:** December 31st, 2024  
- **Annual Summary completion date:** December 31st, 2025

- **IPP Effective Date:** ⚠️  
- **Individual’s day time activity/school:** ⚠️

LIST OF PHYSICIANS FOR ${p.name}

- **Cardiologist:** ⚠️  
- **Dentist:** ⚠️  
- **Neurologist:** ⚠️

LIST OF MEDICATIONS FOR ${p.name}

- **Diagnosis:** ${p.primaryDiagnosis}

| MEDICATIONS | DOSAGE | RATIONALE |  
| --- | --- | --- |

SUPPORT PLAN OUTCOMES

- No progress to date to report as they are new to the home

SUMMARIES OF DAILY LIVING AND ACTIVITIES

- ${p.name} is an individual with ${p.primaryDiagnosis}. No progress to date to report.

HEALTH SUMMARY

- No further information provided.

BEHAVIORAL STATUS

- No information provided.

COMMUNITY INCLUSION

- No information provided.

CHOICES/PREFERENCES

- No information provided.

SAFETY

- No information provided.

TEACHING

- No information provided.

SUMMARY OF SERVICE

- No information provided.
  
–––––

(Note: This is a sample Implementation Plan and Annual Summary for ${p.name} generated for illustrative purposes.)
`;
}

// Call OpenAI API with gpt-3.5-turbo
function callOpenAI(prompt) {
  const url = 'https://api.openai.com/v1/chat/completions';
  
  const maxTokens = 1500;
  const payload = {
    model: "gpt-3.5-turbo",
    messages: [
      { role: "system", content: "You are a helpful assistant that creates comprehensive and clear implementation plans." },
      { role: "user", content: prompt }
    ],
    max_tokens: maxTokens,
    temperature: 0.7
  };
  
  const options = {
    method: 'post',
    contentType: 'application/json',
    headers: { Authorization: `Bearer ${OPENAI_API_KEY}` },
    payload: JSON.stringify(payload),
    muteHttpExceptions: true
  };
  
  const response = UrlFetchApp.fetch(url, options);
  if (response.getResponseCode() !== 200) {
    throw new Error(`OpenAI API error ${response.getResponseCode()}: ${response.getContentText()}`);
  }
  
  return JSON.parse(response.getContentText());
}

// Extract text from OpenAI response
function extractOpenAIText(openAIResponse) {
  if (
    openAIResponse &&
    openAIResponse.choices &&
    openAIResponse.choices.length > 0 &&
    openAIResponse.choices[0].message &&
    openAIResponse.choices[0].message.content
  ) {
    return openAIResponse.choices[0].message.content.trim();
  }
  return '';
}

// Create a Google Doc with the generated plan
function createGoogleDoc(participantName, content) {
  const docName = `Implementation Plan - ${participantName} - ${new Date().toISOString().slice(0,10)}`;
  const doc = DocumentApp.create(docName);
  doc.getBody().setText(content);
  doc.saveAndClose();
}
