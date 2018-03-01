$ docker-compose up -d postgresd
$ docker-compose up hydra-migrate

# Create the client for consent-app
$ hydra clients create --skip-tls-verify \
    --id consent-app \
    --secret consent-secret \
    --name "Consent App Client" \
    --grant-types client_credentials \
    --response-types token \
    --allowed-scopes hydra.consent

$ hydra policies create --skip-tls-verify \
    --actions get,accept,reject \
    --description "Allow consent-app to manage OAuth2 consent requests." \
    --allow \
    --id consent-app-policy \
    --resources "rn:hydra:oauth2:consent:requests:<.*>" \
    --subjects consent-app


# Perform OAuth 2.0 Flow
$ hydra clients create --skip-tls-verify \
    --id some-consumer \
    --secret consumer-secret \
    --grant-types authorization_code,refresh_token,client_credentials,implicit \
    --response-types token,code,id_token \
    --allowed-scopes openid,offline,hydra.clients \
    --callbacks http://localhost:4445/callback

$ hydra policies create --skip-tls-verify \
    --actions get \
    --description "Allow everyone to read the OpenID Connect ID Token public key" \
    --allow \
    --id openid-id_token-policy \
    --resources rn:hydra:keys:hydra.openid.id-token:public \
    --subjects "<.*>"



$ docker-compose up consent-app


# Examplary
$ hydra token user --skip-tls-verify \
    --auth-url http://localhost:4444/oauth2/auth \
    --token-url http://hydra:4444/oauth2/token \
    --id some-consumer \
    --secret consumer-secret \
    --scopes openid,offline,hydra.clients \
    --redirect http://localhost:4445/callback


#####################################################################################
# OathKeeper
#####################################################################################

$ docker-compose up oathkeeper-migrate

docker run --rm -it \
  oryd/oathkeeper:v0.0.29 \
  oathkeeper help serve management


$ echo '{
	"id": "oathkeeper-client-2",
	"client_secret": "secret_key",
	"scope": "hydra.warden hydra.keys.* hydra.introspect",
	"grant_types": ["client_credentials"],
	"response_types": ["token"]
}' > client.json

$ echo '{
	"subjects": ["oathkeeper-client-2"],
	"effect": "allow",
	"resources": [
		"rn:hydra:keys:oathkeeper:id-token-2<.*>",
		"rn:hydra:warden:<.*>",
		"rn:hydra:oauth2:tokens"
	],
	"actions": ["decide","get","create","introspect","update","delete"]
}' > policy.json

$ hydra clients import client.json
$ hydra policies import policy.json
