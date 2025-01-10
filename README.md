Integrating Sanity with Next.js: A Guide to Data Import and Environment Setup

In this blog post, we’ll walk through the process of integrating Sanity with an existing Next.js project, focusing on setting up environmental variables, creating a Sanity schema, and importing data from an external API. This guide assumes you already have a Next.js project set up and Sanity installed.

Table of Contents
1. Setting Up Environment Variables
2. Obtaining Sanity Project ID and API Token
3. Creating the Sanity Schema
4. Setting Up the Data Import Script
5. Running the Import Script

1. Setting Up Environment Variables
First, let’s set up our environment variables. Create a `.env.local` file in your project root if it doesn't already exist. Add the following variables:

NEXT_PUBLIC_SANITY_PROJECT_ID=your_project_id
NEXT_PUBLIC_SANITY_DATASET=production
SANITY_API_TOKEN=your_sanity_token
Remember, variables prefixed with NEXT_PUBLIC_ will be exposed to the browser, so be cautious about what you prefix.

2. Obtaining Sanity Project ID and API Token
Project ID
To find your Sanity project ID:

Log in to your Sanity account at https://www.sanity.io/manage
2. Select your project
3. In the project dashboard, you’ll see the project ID listed

Use this ID for the `NEXT_PUBLIC_SANITY_PROJECT_ID` in your `.env.local` file.

API Token

To generate a Sanity API token:

1. Go to https://www.sanity.io/manage and select your project
2. Navigate to the “API" tab
3. Under "Tokens," click "Add API token"
4. Give your token a name and select the appropriate permissions (usually "Editor" for full read/write access)
5. Copy the generated token




Use this token for the `SANITY_API_TOKEN` in your `.env.local` file.

3. Creating the Sanity Schema
Now, let’s create a schema for our products. In your Sanity schema folder (usually `sanity/schemaTypes`), create a new file called `product.ts`:

export default {
  name: 'product',
  type: 'document',
  title: 'Product',
  fields: [
    {
      name: 'name',
      type: 'string',
      title: 'Product Name',
    },
    {
      name: 'description',
      type: 'string',
      title: 'Description'
    },
    {
      name: 'price',
      type: 'number',
      title: 'Product Price',
    },
    {
      name: 'discountPercentage',
      type: 'number',
      title: 'Discount Percentage',
    },
    {
      name: 'priceWithoutDiscount',
      type: 'number',
      title: 'Price Without Discount',
      description: 'Original price before discount'
    },
    {
      name:'rating',
      type:'number',
      title:'Rating',
      description:'Rating of the product'
    },
    {
      name: 'ratingCount',
      type: 'number',
      title: 'Rating Count',
      description: 'Number of ratings'
    },
    {
      name: 'tags',
      type: 'array',
      title: 'Tags',
      of: [{ type: 'string' }],
      options: {
        layout: 'tags'
      },
      description: 'Add tags like "new arrival", "bestseller", etc.'
    },
    {
      name: 'sizes',
      type: 'array',
      title: 'Sizes',
      of: [{ type: 'string' }],
      options: {
        layout: 'tags'
      },
      description: 'Add sizes like S , M , L , XL , XXL'
    },
    {
      name: 'image',
      type: 'image',
      title: 'Product Image',
      options: {
        hotspot: true // Enables cropping and focal point selection
      }
    }
  ]
};
Then, update your `sanity/schemaTypes/index.ts` file to include the new product schema:

import { type SchemaTypeDefinition } from 'sanity'
import product from './product'

export const schema: { types: SchemaTypeDefinition[] } = {
  types: [product],
}
4. Setting Up the Data Import Script

Now, let’s create a script to import data from an external API into Sanity. Create a new file `scripts/importSanityData.mjs` in your project root:

import { createClient } from '@sanity/client'
import axios from 'axios'
import dotenv from 'dotenv'
import { fileURLToPath } from 'url'
import path from 'path'

// Load environment variables from .env.local
const __filename = fileURLToPath(import.meta.url)
const __dirname = path.dirname(__filename)
dotenv.config({ path: path.resolve(__dirname, '../.env.local') })
// Create Sanity client
const client = createClient({
  projectId: process.env.NEXT_PUBLIC_SANITY_PROJECT_ID,
  dataset: process.env.NEXT_PUBLIC_SANITY_DATASET,
  useCdn: false,
  token: process.env.SANITY_API_TOKEN,
  apiVersion: '2021-08-31'
})
async function uploadImageToSanity(imageUrl) {
  try {
    console.log(`Uploading image: ${imageUrl}`)
    const response = await axios.get(imageUrl, { responseType: 'arraybuffer' })
    const buffer = Buffer.from(response.data)
    const asset = await client.assets.upload('image', buffer, {
      filename: imageUrl.split('/').pop()
    })
    console.log(`Image uploaded successfully: ${asset._id}`)
    return asset._id
  } catch (error) {
    console.error('Failed to upload image:', imageUrl, error)
    return null
  }
}
async function importData() {
  try {
    console.log('Fetching products from API...')
    const response = await axios.get('https://fakestoreapi.com/products')
    const products = response.data
    console.log(`Fetched ${products.length} products`)
    for (const product of products) {
      console.log(`Processing product: ${product.title}`)
      let imageRef = null
      if (product.image) {
        imageRef = await uploadImageToSanity(product.image)
      }
      const sanityProduct = {
        _type: 'product',
        name: product.title,
        description: product.description,
        price: product.price,
        discountPercentage: 0,
        priceWithoutDiscount: product.price,
        rating: product.rating?.rate || 0,
        ratingCount: product.rating?.count || 0,
        tags: product.category ? [product.category] : [],
        sizes: [],
        image: imageRef ? {
          _type: 'image',
          asset: {
            _type: 'reference',
            _ref: imageRef,
          },
        } : undefined,
      }
      console.log('Uploading product to Sanity:', sanityProduct.name)
      const result = await client.create(sanityProduct)
      console.log(`Product uploaded successfully: ${result._id}`)
    }
    console.log('Data import completed successfully!')
  } catch (error) {
    console.error('Error importing data:', error)
  }
}
importData()
Now, let’s install the necessary packages. Run the following command in your terminal:

npm install @sanity/client axios dotenv
5. Running the Import Script

To run the import script, we need to add a new script to our `package.json` file. Open your `package.json` and add the following to the `"scripts"` section:

"scripts": {
  "dev": "next dev --turbopack",
  "build": "next build",
  "start": "next start",
  "lint": "next lint",
  "import-data": "node scripts/importSanityData.mjs"
}
Now you can run the import script using:

npm run import-data
This script will fetch products from the FakeStoreAPI, upload any associated images to Sanity’s asset store, and then create new product documents in your Sanity dataset.

Conclusion
By following this guide, you’ve learned how to set up environment variables for your Sanity integration, obtain your Sanity project ID and API token, create a custom schema for products, and import data from an external API into your Sanity content lake.

Remember to keep your `.env.local` file secure and never commit it to your repository. For production deployments, set these environment variables in your hosting platform instead of including them in your code.

This integration allows you to leverage Sanity’s powerful content management capabilities while working with external data sources, providing a flexible and scalable solution for your Next.js project.
