<script setup lang="ts">
import {computed, ref} from 'vue'
import {useField, useForm} from 'vee-validate'
import {VatType} from '@/components/form/enum/VatType'

const {handleSubmit, handleReset} = useForm({
  validationSchema: {
    descriptionSchema(value: string) {
      if (!value) return 'Text is required'

      if (value?.length > 255) return 'You can’t enter more than 255 characters'

      return true
    },
    ifSendConfirmationSchema(value: boolean | undefined) {
      if (typeof value === 'undefined') return 'Text is required'

      return true
    },
    vatValueSchema(value: string) {
      if (!value) return 'Text is required'

      return true
    },
    priceNettoSchema(value: string) {
      if (!value) return 'Text is required'

      if(value.match(/^-?\d*(\.\d+)?$/)) return true

      const normalizedValue = value.replace(',', '.')
      return normalizedValue.match(/^-?\d*(\.\d+)?$/) ? true : 'Please, input a valid number'
    }
  }
})
const description = useField('descriptionSchema')
const ifSendConfirmation = useField('ifSendConfirmationSchema')
const vatValue = useField('vatValueSchema')
const priceNetto = useField('priceNettoSchema')

const ifDescriptionFieldActive = ref<boolean>(false)
const descriptionLengthUserAbleToAdd = computed(() => {
  const ableToAdd = 255 - (description.value.value?.length || 0)

  return (ableToAdd < 0) ? 0 : ableToAdd
})

const vatVariants:VatType[] = [VatType.Vat_19, VatType.Vat_21, VatType.Vat_23, VatType.Vat_25]
const vatNumberValue = computed(() => {
  const vatStringToNumberValueMap = {
    [VatType.Vat_19]: 19,
    [VatType.Vat_21]: 21,
    [VatType.Vat_23]: 23,
    [VatType.Vat_25]: 25,
  }

  return vatStringToNumberValueMap[vatValue.value.value] || 0
})

const priceBrutto = computed(() => {
  if(vatNumberValue && priceNetto.value.value && !priceNetto.errorMessage.value) {
    const normalizeNetto = priceNetto.value.value.replace(',', '.')

    return Number(normalizeNetto) * (1 + vatNumberValue.value/100)
  }

  return ''
})

const isFormSent = ref<boolean>(false)
const formErrorMessage = ref<string>('')
const ifDataSending = ref<boolean>(false)
const submit = handleSubmit(async () => {
  try {
    ifDataSending.value = true

    const response = await fetch('https://httpbin.org/post', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        description: description.value.value,
        sendConfirmation: ifSendConfirmation.value.value,
        vat: vatValue.value.value,
        priceNettoEur: priceNetto.value.value.replace(',', '.'),
        priceBruttoEur: priceBrutto.value,
      }),
    })

    if (response.ok) {
      handleFormSuccess()
    } else {
      handleFormError(response.status)
    }
  } catch (error) {
    ifDataSending.value = false
    handleFormError()
    console.error('Error during form sending:', error)
  }

  ifDataSending.value = false
})

const handleFormSuccess = () => {
  isFormSent.value = true
  formErrorMessage.value = ''
}

const handleFormError = (status?: number) => {
  isFormSent.value = true

  switch (status) {
    case 404:
      formErrorMessage.value = 'Page Not found'
      throw new Error('404, Not found')
    case 500:
      formErrorMessage.value = 'Internal server error'
      throw new Error('500, internal server error')
    default:
      formErrorMessage.value = 'Something went wrong'
      throw new Error(String(status))
  }
}

const resetForm = () => {
  handleReset()
  isFormSent.value = false
  formErrorMessage.value = ''
}
</script>

<template>
  <div class="page-wrapper">
    <div class="form-container" v-if="!isFormSent">
      <div class="form-header">Send your data</div>
      <form>
        <div class="form-item-container">
          <div class="description-wrapper">
            <v-textarea
              label="Description"
              variant="outlined"
              v-model="description.value.value"
              :error-messages="description.errorMessage.value"
              @focus="ifDescriptionFieldActive = true"
              @blur="ifDescriptionFieldActive = false"
              no-resize
            ></v-textarea>
            <div class="description-additional-info-container" v-if="ifDescriptionFieldActive">
              {{descriptionLengthUserAbleToAdd}}
            </div>
          </div>
        </div>
        <div class="form-item-container">
          <div class="confirmation-label" :class="{'label-error': ifSendConfirmation.errorMessage.value}">Send confirmation</div>
          <v-radio-group
            inline
            v-model="ifSendConfirmation.value.value"
            :error-messages="ifSendConfirmation.errorMessage.value"
          >
            <v-radio label="Yes" class="radio-btn" :value="true"></v-radio>
            <v-radio label="No" class="radio-btn" :value="false"></v-radio>
          </v-radio-group>
        </div>
        <div class="form-item-container">
          <v-select
            label="Choose VAT"
            :items="vatVariants"
            variant="outlined"
            v-model="vatValue.value.value"
            :error-messages="vatValue.errorMessage.value"
          ></v-select>
        </div>
        <div class="form-item-container">
          <v-text-field
            label="Price netto EUR"
            variant="outlined"
            v-model="priceNetto.value.value"
            :error-messages="priceNetto.errorMessage.value"
            :disabled="!vatValue.value.value"
          ></v-text-field>
        </div>
        <div class="form-item-container">
          <v-text-field
            label="Price brutto EUR"
            variant="outlined"
            v-model="priceBrutto"
            readonly
          ></v-text-field>
        </div>
        <div class="form-item-container">
          <v-btn
            @click="submit"
            :loading="ifDataSending"
          >
            Submit
          </v-btn>
        </div>
      </form>
    </div>
    <div class="form-sent-message-container" :class="[formErrorMessage ? 'fail' : 'success']" v-else>
      <div class="form-sent-svg-container">
        <svg
          fill="white"
          width="120px"
          height="120px"
          viewBox="0 0 24 24"
          xmlns="http://www.w3.org/2000/svg"
          v-if="!formErrorMessage"
        >
          <path fill-rule="evenodd" d="M12,2 C17.5228475,2 22,6.4771525 22,12 C22,17.5228475 17.5228475,22 12,22 C6.4771525,22 2,17.5228475 2,12 C2,6.4771525 6.4771525,2 12,2 Z M12,4 C7.581722,4 4,7.581722 4,12 C4,16.418278 7.581722,20 12,20 C16.418278,20 20,16.418278 20,12 C20,7.581722 16.418278,4 12,4 Z M15.2928932,8.29289322 L10,13.5857864 L8.70710678,12.2928932 C8.31658249,11.9023689 7.68341751,11.9023689 7.29289322,12.2928932 C6.90236893,12.6834175 6.90236893,13.3165825 7.29289322,13.7071068 L9.29289322,15.7071068 C9.68341751,16.0976311 10.3165825,16.0976311 10.7071068,15.7071068 L16.7071068,9.70710678 C17.0976311,9.31658249 17.0976311,8.68341751 16.7071068,8.29289322 C16.3165825,7.90236893 15.6834175,7.90236893 15.2928932,8.29289322 Z"/>
        </svg>
        <svg
          fill="white"
          width="120px"
          height="120px"
          viewBox="-1.7 0 20.4 20.4"
          xmlns="http://www.w3.org/2000/svg"
          class="cf-icon-svg"
          v-else
        >
          <path d="M16.417 10.283A7.917 7.917 0 1 1 8.5 2.366a7.916 7.916 0 0 1 7.917 7.917zm-6.804.01 3.032-3.033a.792.792 0 0 0-1.12-1.12L8.494 9.173 5.46 6.14a.792.792 0 0 0-1.12 1.12l3.034 3.033-3.033 3.033a.792.792 0 0 0 1.12 1.119l3.032-3.033 3.033 3.033a.792.792 0 0 0 1.12-1.12z"/>
        </svg>
      </div>
      <div class="form-sent-message">{{formErrorMessage ? formErrorMessage : 'Form sent successfully!'}}</div>
      <v-btn
        @click="resetForm"
      >
        Sent new data
      </v-btn>
    </div>
  </div>
</template>

<style scoped lang="scss">
  .form-container {
    max-width: 600px;
    padding: 15px;
    margin: 40px auto 0 auto;

    .form-header{
      display: flex;
      justify-content: center;
      font-size: 24px;
      margin-bottom: 12px;
    }

    .form-item-container{
      margin-bottom: 12px;

      .description-wrapper{
        position: relative;

        .description-additional-info-container{
          position: absolute;
          bottom: 0;
          right: 0;
          font-size: 12px;
        }
      }

      .confirmation-label{
        font-size: 14px;
        margin-bottom: 8px;

        &.label-error{
          color: #B00020;
        }
      }

      .radio-btn{
        height: 20px;
      }
    }
  }

  .form-sent-message-container{
    margin: 40px auto;
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    max-width: 460px;
    height: 350px;
    border-radius: 15px;

    /* shadow */
    -webkit-box-shadow: 8px 8px 42px -20px rgba(66, 68, 90, 1);
    -moz-box-shadow: 8px 8px 42px -20px rgba(66, 68, 90, 1);
    box-shadow: 8px 8px 42px -20px rgba(66, 68, 90, 1);

    &.success{
      background: green;
    }

    &.fail{
      background: lightcoral;
    }

    .form-sent-svg-container{
      margin-bottom: 12px;
    }

    .form-sent-message{
      margin-bottom: 12px;
      font-size: 20px;
      font-weight: bold;
      color: white;
    }
  }

  @media (min-width: 650px) {
    .form-container{
      background-color: white;
      border-radius: 15px;

      /* shadow */
      -webkit-box-shadow: 8px 8px 42px -20px rgba(66, 68, 90, 1);
      -moz-box-shadow: 8px 8px 42px -20px rgba(66, 68, 90, 1);
      box-shadow: 8px 8px 42px -20px rgba(66, 68, 90, 1);
    }
  }

  @media (max-width: 650px) {
    .form-container{
      margin: 0 auto;
    }
  }

  @media (max-width: 520px) {
    .form-sent-message-container{
      width: 90vw;
    }
  }
</style>
