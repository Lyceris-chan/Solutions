## T-rex Miner WebUI setup instructions when using SaladBind

- Open command prompt and navigate to the directory where the miner executable is.
Run t-rex --api-generate-key <your_password>.
- Make a config.txt in the data folder of saladbind
- Copy the raw config from t-rex in there (check raw config on the webui)
- Open saladbind and use the following arguments:
--api-key + -a ++ -c +++

replace + with your api key
replace ++ with your algorithm (ethash)
replace +++ with the config file you made earlier (path/filename) u get what i mean

