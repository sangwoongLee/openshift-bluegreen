import java.text.SimpleDateFormat;

def funCreateImageTag(selector, namespaces){
    
    sh "oc get istag -l='${selector}' -n ${namespaces} -o json > result.json"
    def myJson = readFile('result.json');
    def myObject = readJSON text: myJson;
    def items = myObject.items
    def choices = ""
    
    items.each { item ->
        choices += "${item.metadata.name}\n"
    }    
    return choices
}

def selectedImageTag = "";

node {
  // Blue/Green Deployment into Production
  // -------------------------------------
  def project  = "prd-sample-dashboard"
  def dest     = "dashboard-green"
  def active   = ""

  stage('Determine Deployment color') {

    sh "oc get route dashboard -n ${project} -o jsonpath='{ .spec.to.name }' > activesvc.txt"

    // Determine currently active Service
    active = readFile('activesvc.txt').trim()
    if (active == "dashboard-green") {
      dest = "dashboard-blue"
    }
    echo "Active svc: " + active
    echo "Dest svc:   " + dest
  }

  stage('Select new version') {
        
    script {
          def choices = funCreateImageTag("app=sample-dashboard", "prd-sample-dashboard")
          env.userInput = input(id: 'userInput', message: 'Select Image?',
              parameters: [[$class: 'ChoiceParameterDefinition', defaultValue: 'strDef', 
              description:'Select-Promotion-Image', name:'selectedImage', choices: choices]
          ])

          selectedImageTag = "${env.userInput}"
          echo "selectedImage: "+ selectedImageTag
    }
  }

  stage('Deploy new Version') {
    echo "Deploying to ${dest}"
   
    def dockerRepo = "docker-registry.default.svc:5000/" + project
    def destDockerImage = dockerRepo + "/" + selectedImageTag
    
    sh 'oc patch dc -n ' + project +' '+ dest +' -p \'{"spec":{"template" : {"spec": {"containers":[{"name":"dashboard", "image":"temp:latest"} ]}}}}\''
    sh 'oc patch dc -n ' + project +' '+ dest +' -p \'{"spec":{"template" : {"spec": {"containers":[{"name":"dashboard", "image":"' + destDockerImage + '"} ]}}}}\''
    
    openshiftDeploy depCfg: dest, namespace: project, verbose: 'true', waitTime: '', waitUnit: 'sec'
  }
  
  stage('Testing new Version') {
    openshiftVerifyDeployment depCfg: dest, namespace: project, replicaCount: '1', verbose: 'false', verifyReplicaCount: 'true', waitTime: '', waitUnit: 'sec'
    //openshiftVerifyService namespace: project, svcName: dest, verbose: 'false'
  }
  
  stage('Switch over to new Version') {
    input "Switch Production?"
    sh 'oc patch route -n ' + project + ' dashboard -p \'{"spec":{"to":{"name":"' + dest + '"}}}\''
    sh 'oc get route -n ' + project + ' dashboard > oc_out.txt'
    oc_out = readFile('oc_out.txt')
    echo "Current dashboard configuration: " + oc_out
  }
}
