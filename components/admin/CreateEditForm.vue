<template>
  <v-form @submit.prevent="handleSubmit()">
    <h1 class="text-center">Create a new bag.</h1>
    <section class="data">
      <v-text-field
        v-model="bag.name.en"
        outlined
        label="bag name (EN)"
        placeholder="ENGLISH"
      >
      </v-text-field>
      <v-text-field
        v-model="bag.name.it"
        outlined
        label="bag name (IT)"
        placeholder="ITALIAN"
      >
      </v-text-field>
    </section>

    <section class="data">
      <v-textarea
        v-model="bag.description.en"
        label="description(en)"
        placeholder="ENGLISH"
      ></v-textarea>
      <v-textarea
        v-model="bag.description.it"
        label="description(it)"
        placeholder="ITALIAN"
      ></v-textarea>
    </section>

    <section class="data">
      <v-file-input
        accept="image/*"
        label="Main image"
        @change="(e) => setFile(e)"
      ></v-file-input>
      <v-dialog v-model="dialog">
        <template #activator="{ on, attrs }">
          <v-btn v-bind="attrs" color="warning" v-on="on" @click="dialog = true"
            >Add variant</v-btn
          >
        </template>
        <VariantForm @add="(payload) => handleAdd(payload)" />
      </v-dialog>
    </section>

    <section v-if="bag.variants.length > 0" class="data">
      <v-row>
        <v-col
          v-for="item in bag.variants"
          :key="item.i"
          cols="12"
          class="d-flex flex-row justify-space-around"
        >
          <DisplayVariant
            :variant="item"
            @deleteVariant="(payload) => deleteVariant(payload)"
          />
        </v-col>
      </v-row>
    </section>
    <v-btn
      class="mt-15"
      type="submit"
      :loading="isLoading"
      :disabled="!isValid"
      color="primary"
      block
      >submit</v-btn
    >
  </v-form>
</template>

<script>
// import axios from 'axios'
import { createVariantProduct } from '../../composables/stripeHandlers.js'
import {
  uploadDefaultImage,
  uploadVariantImages,
  // eslint-disable-next-line no-unused-vars
  saveToDB,
} from '../../composables/uploadHandlers.js'
import VariantForm from './VariantForm'
import DisplayVariant from './DisplayVariant.vue'
export default {
  components: { VariantForm, DisplayVariant },
  data() {
    return {
      dialog: false,
      isLoading: false,

      cloudinaryFolderName: '',
      bag: {
        name: {
          en: '',
          it: '',
        },
        description: {
          en: '',
          it: '',
        },
        variants: [],
        price: {
          amount: null,
          currency: 'EUR',
        },
        defaultImage: null,
      },
    }
  },
  computed: {
    isValid() {
      return true // uncomment for testing
      // const { name, description, price, variants } = this.bag
      // const { en, it } = description
      // return (
      //   en &&
      //   it &&
      //   name.en &&
      //   name.it &&
      //   price.amount > 0 &&
      //   // eslint-disable-next-line no-self-compare
      //   parseInt(price.amount) === parseInt(price.amount) &&
      //   variants.length > 0 &&
      //   variants.images.length > 0
      // )
    },
  },

  methods: {
    async handleSubmit() {
      this.isLoading = true
      try {
        this.cloudinaryFolderName = await uploadDefaultImage(this.bag)

        if (this.bag.variants.length > 0) {
          await uploadVariantImages(this.bag, this.cloudinaryFolderName)
          for (let i = 0; i < this.bag.variants.length; i++) {
            this.bag.variants[i].stripeProductId = await createVariantProduct(
              this.bag.variants[i],
              this.bag
            )
          }
        }

        await saveToDB(this.bag)
      } catch (error) {
        alert(error)
      } finally {
        this.isLoading = false
        this.bag = {
          name: {
            en: '',
            it: '',
          },
          description: {
            en: '',
            it: '',
          },
          variants: [],
          price: {
            amount: null,
            currency: 'EUR',
          },
          defaultImage: null,
        }
        location.reload()
      }
    },

    setFile(e) {
      try {
        this.bag.defaultImage = e
      } catch (error) {
        alert(error)
      }
    },

    handleAdd(payload) {
      this.bag.variants.push(payload)
      this.dialog = false
    },
    deleteVariant(payload) {
      this.bag.variants = this.bag.variants.filter((e) => e !== payload)
    },
  },
}
</script>

<style scoped>
.data {
  display: flex;
  flex-direction: row;
  justify-content: space-around;
}
.data > * {
  margin: 1rem;
}
</style>