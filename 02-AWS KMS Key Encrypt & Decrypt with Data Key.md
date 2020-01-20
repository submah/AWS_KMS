# AWS KMS Key Encrypt & Decrypt with Data Key
Using KMS basic encryption is easy, but it comes with few drawbacks.

Encrypting a significant amount of data is expensive as you have to transmit all your data over the wire in order to encrypt it on Amazon’s server.

Transferring data over a network could cause potential security breaches and lead to an unauthorised disclosure of, or access to your data.

The built-in 4KB limitation prevents you from encrypting large files. You could chunk the data up and reassemble it later during decryption, but rather than doing that let us have a look how we can do better by applying Envelope encryption.

Envelope Encryption is a practice of encrypting plaintext data with a Unique Data Key, and then encrypting the Data Key with a key encryption key (KEK).


# 1. Create Customer Master Key(CMK)
Lets create a new Customer Master Key that will be used to encrypt data.

aws kms create-key

# Sample output
{
    "KeyMetadata": {
        "AWSAccountId": "123411223344",
        "KeyId":"6fa6043b-2fd4-433b-83a5-3f4193d7d1a6",
        "Arn":"arn:aws:kms:us-east-1:123411223344:key/6fa6043b-2fd4-433b-83a5-3f4193d7d1a6",
        "CreationDate": 1547913852.892,
        "Enabled": true,
        "Description": "",
        "KeyUsage": "ENCRYPT_DECRYPT",
        "KeyState": "Enabled",
        "Origin": "AWS_KMS",
        "KeyManager": "CUSTOMER"
    }
}

# Create an Key Alias
aws kms create-alias \
    --alias-name "alias/kms-data-key-demo" \
    --target-key-id "cf84167d-44cb-4b07-b277-ae1ea21dbe4d"
    
  # 2. Create a data key
  
  Now, with our new CMK lets generate a Data Key. This will give us a CiphertextBlob which is base64 encoded. The blob contains the encrypted data key and also the meta-data about which CMK was used during data key creation & it will allow us to retrieve the plaintext key later on decryption.

# Generate the key and store it in a hidden directory called `.key`
mkdir -p "./.key" "encrypted_data" "decrypted_data"
aws kms generate-data-key \
    --key-id "alias/kms-data-key-demo" \
    --key-spec "AES_256" \
    --output text \
    --query CiphertextBlob | base64 --decode > "./.key/encrypted_data_key"
    
   
# Extract Plaintext Data Key from CiphertextBlob
 To encrypt your data, we will need the _data key_ in the plaintext format. 
 _You can store the output in a file, if you want to use for multiple opertaions or in memory as shown below_
 
 plaintext_data_key=$(aws kms decrypt \
                        --ciphertext-blob \
                        fileb://./.key/encrypted_data_key \
                        --output text \
                        --query Plaintext
                        )
                        
# 3. Encrypt Data with Data Key
ets begin the encryption of our confidential data. You can use any tool for this, here, I will be using OpenSSL. The encrypted output is stored in a file encrypted_data.txt

openssl enc -e -aes256 \
    -k "${plaintext_data_key}" \
    -in "confidential_data.txt" \
    -out "./encrypted_data/confidential_data.txt.encrypted"

# In case you want to store the key in a file and encrypt,
openssl enc -e -aes256 \
    -k fileb://./key/plaintext_data_key \
    -in "confidential_data.txt" \
    -out "./encrypted_data/confidential_data.txt.encrypted"
   
# 4. Decrypting the data with Data Key
Decrypting data is the most straightforward, assuming you have stored the encrypted data key securely and able to access it when required. In this demo, we have stored it under .key directory. We will need the plaintext copy of the data key.

# Generate the plaintext copy of the data key and store in memory
plaintext_data_key=$(aws kms decrypt \
                        --ciphertext-blob \
                        fileb://./.key/encrypted_data_key \
                        --output text \
                        --query Plaintext
                        )

# Decrypt the data
openssl enc -d -aes256 \
    -k "${plaintext_data_key}" \
    -in "./encrypted_data/confidential_data.txt.encrypted" \
    -out "./decrypted_data/confidential_data.txt.decrypted"

# In case you want to store the key in a file and encrypt,
openssl enc -d -aes256 \
    -kfile "./.key/plaintext_data_key" \
    -in "./encrypted_data/confidential_data.txt.encrypted" \
    -out "./decrypted_data/confidential_data.txt.decrypted"
    
    
