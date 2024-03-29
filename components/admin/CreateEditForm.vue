<template>
  <v-form @submit.prevent="handleSubmit()">
    <h1 class="text-center">
      {{ product ? 'Edit the bag:' : 'Create a new bag:' }}
    </h1>
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
        v-if="!isImageUploaded"
        accept="image/*"
        label="Main cover image (will be used for featured section)"
        @change="(e) => setFile(e)"
      ></v-file-input>
      <div v-else class="d-flex flex-column">
        <h3 class="text-center">default image</h3>
        <v-img width="200" :src="bag.defaultImage"></v-img>
        <v-btn class="mx-auto my-2" color="error" @click="bag.defaultImage = ''"
          >delete image</v-btn
        >
      </div>
      <v-dialog v-model="dialog">
        <template #activator="{ on, attrs }">
          <v-btn
            v-bind="attrs"
            color="warning"
            class="my-auto"
            v-on="on"
            @click="dialog = true"
            >Add variant</v-btn
          >
        </template>
        <VariantForm
          :product="variant"
          :edit="edit"
          @add="(payload) => handleAdd(payload)"
        />
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
            :edit="edit"
            @deleteVariant="(payload) => deleteVariant(payload)"
            @editVariant="(payload) => editVariant(payload)"
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
      >{{ edit ? 'save' : 'create' }}</v-btn
    >
  </v-form>
</template>

<script>
/* eslint-disable no-void */

import { createVariantProduct } from '../../composables/stripeHandlers.js'
import {
  uploadDefaultImage,
  uploadVariantImages,
  saveToDB,
} from '../../composables/uploadHandlers.js'
import VariantForm from './VariantForm'
import DisplayVariant from './DisplayVariant.vue'
export default {
  components: { VariantForm, DisplayVariant },
  props: {
    product: {
      type: Object,
      default: null,
    },
    edit: {
      type: Boolean,
      default: false,
    },
  },
  data() {
    return {
      dialog: false,
      isLoading: false,
      variant: {
        color: '',
        colorName: { en: '', it: '' },
        price: null,
        stock: null,
        images: [],
        stripeProductId: '',
      },
      cloudinaryFolderName: '',
      bag: this.product
        ? this.product
        : {
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
      // return true // uncomment for testing
      const { name, description, variants } = this.bag
      const { en, it } = description
      return en && it && name.en && name.it && variants.length > 0
    },
    isImageUploaded() {
      return typeof this.bag.defaultImage === 'string' &&
        this.bag.defaultImage.length > 0
        ? !!this.bag.defaultImage.includes('https')
        : false
      // return this.bag.defaultImage || this.bag.defaultImage.includes('https')
    },
  },
  mounted() {
    //
  },

  methods: {
    async handleSubmit() {
      // if (!this.edit) {
      this.isLoading = true
      try {
        if (this.isImageUploaded) {
          this.cloudinaryFolderName = this.bag.defaultImage.split('/')[8]
        } else if (this.edit) {
          // editing
          // await this.deleteDefaultImage()
          this.cloudinaryFolderName = await uploadDefaultImage(this.bag, false)
        } else {
          // creating
          this.cloudinaryFolderName = await uploadDefaultImage(this.bag, true)
        }

        if (this.bag.variants.length > 0) {
          if (this.edit) {
            const { data } = await this.$axios.get(
              `/api/v1/bags/${this.product._id}`
            )
            // fetch the bag from db and compare with local data
            if (
              JSON.stringify(data.variants) !==
              JSON.stringify(this.bag.variants)
            ) {
              // console.log('local variants are different from remote ones') // TODO: upload files
              // console.log('local', JSON.stringify(this.bag.variants))
              // console.log('remote', JSON.stringify(data.variants))
            } else {
              // console.log('local variants are the same as remote')
              // console.log('local', JSON.stringify(this.bag.variants))
              // console.log('remote', JSON.stringify(data.variants))
            }
          } else {
            await uploadVariantImages(this.bag, this.cloudinaryFolderName)
            for (let i = 0; i < this.bag.variants.length; i++) {
              this.bag.variants[i].stripeProductId = await createVariantProduct(
                this.bag.variants[i],
                this.bag
              )
            }
          }
        }

        await saveToDB(this.bag, this.product)
      } catch (error) {
        alert(error)
        // console.log(error)
      } finally {
        this.isLoading = false
        if (!this.edit) {
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
        }
      }
      // } else {
      //   console.log('bag to save:', this.bag)
      // }
    },

    setFile(e) {
      try {
        this.bag.defaultImage = e
      } catch (error) {
        alert(error)
      }
    },

    handleAdd(payload) {
      if (this.edit) {
        this.bag.variants = this.bag.variants.filter(
          (e) => e._id !== payload._id
        )
        this.bag.variants.push(payload)
        this.dialog = false
      } else {
        this.bag.variants.push(payload)
        this.dialog = false
      }
    },
    deleteVariant(payload) {
      this.bag.variants = this.bag.variants.filter((e) => e !== payload)
    },

    editVariant(payload) {
      this.variant = null
      this.variant = payload
      this.dialog = true
    },

    async deleteDefaultImage() {
      const imageId = this.bag.defaultImage.split('/')
      await this.$axios.post(
        '/api/files/delete-image',
        {
          public_id:
            imageId[7] + '/' + imageId[8] + '/' + imageId[9].split('.')[0],
        },
        {
          headers: {
            Authorization: `Bearer ${this.$store.state.token}`,
          },
        }
      )
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
