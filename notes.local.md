(.venv) verspgr@2Q0TPN3:~/mustry-academy/cicd-lab-03-github-actions$ yamllint -c .yamllint.yml . # YAML style + syntax
./docker-compose.yml
  21:35     error    trailing spaces  (trailing-spaces)

(.venv) verspgr@2Q0TPN3:~/mustry-academy/cicd-lab-03-github-actions$ 

(.venv) verspgr@2Q0TPN3:~/mustry-academy/cicd-lab-03-github-actions$ shellcheck scripts/*.sh     # shell-script bugs

In scripts/scan.sh line 53:
    $URL/data/api/v1/scan/$1 || echo 000
    ^--^ SC2086 (info): Double quote to prevent globbing and word splitting.
                          ^-- SC2086 (info): Double quote to prevent globbing and word splitting.

Did you mean: 
    "$URL"/data/api/v1/scan/"$1" || echo 000

For more information:
  https://www.shellcheck.net/wiki/SC2086 -- Double quote to prevent globbing ...
(.venv) verspgr@2Q0TPN3:~/mustry-academy/cicd-lab-03-github-actions$ 