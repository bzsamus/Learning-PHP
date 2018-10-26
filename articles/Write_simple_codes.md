# 1. Make use of conditional operators

## Elvis operator

```
if ($_POST['userInput']) {
    $input = $_POST['userInput'];
} else {
    $input = $defaultValue;
}

$input = $_POST['userInput'] ? $_POST['userInput'] : $defaultValue;

$input = $_POST['userInput'] ?: $defaultValue;
```

## Null coalesce operator

```
$input = isset($_POST['userInput']) ? $_POST['userInput'] : $defaultValue;

$input = $_POST['userInput'] ?? $defaultValue;

$input = $_POST['userInput1'] ?? $_POST['userInput2'] ?? $_POST['userInput3'] ?? $defaultValue;
```



# 2. Low level of indentation

Original code

```
public function filterReport($postData)
{
    if (!empty($postData)) {
        
        // FILTER ACCOUNT(s)
        if (isset($postData['accounts']) AND !empty($postData['accounts'])) {
            $or = array();
            foreach ($postData['accounts'] as $k => $account) {
                $or[] = 'ml.account='.$account;
            }
            $conditions[] .= '('. implode(' OR ', $or) . ')';
        } else {
            if ($this->highest_user_level == STANDARD) {
                $conditions[] .= 'ml.account ='.$this->account->getId();
            }
        }

        // FILTER USER(s)
        if ($this->highest_user_level == STANDARD) {
            $conditions[] .= 'ml.user ='.$this->user->getId();
        } else {
            if (isset($postData['users']) AND !empty($postData['users'])) {
                $accountuser = $this->doctrine->em->getRepository('models\entities\Accountuser')->findOneBy(array('user' => $postData['report_user'], 'account' => $postData['accounts'][0], 'deleted' => '0'));
                if ($accountuser && $accountuser->getUserlevel()->getId() == STANDARD) {
                    $conditions[] .= 'ml.user ='.$postData['report_user'];
                } else {
                    if (count($postData['users']) != 1 || $postData['users'][0] != $postData['report_user']) {
                        $or = array();
                        foreach ($postData['users'] as $k => $user) {
                            $or[] = 'ml.user='.$user;
                        }
                        $conditions[] .= '('. implode(' OR ', $or) . ')';
                    }
                }
            }
        }
    }
}
```

We can immediately make filterReport function simplier by refactoring using functions. Now filterReport function only has one level of indentation.

```
public function filterReport($postData)
{
    if (!empty($postData)) {
        $this->filterReportAccount();
        $this->filterReportUser();
    }
}
```

Refactor filterReportAccount function
```
protected function filterReportAccount()
{
    // FILTER ACCOUNT(s)
    if (isset($postData['accounts']) && !empty($postData['accounts'])) {
        $or = [];
        array_map(
            function ($account) use ($or) {
                $or[] = sprintf('ml.account=%s', $account);
            },
            $postData['accounts']
        );
        $conditions[] .= '('. implode(' OR ', $or) . ')';
    } elseif ($this->highest_user_level == STANDARD) {
        $conditions[] .= 'ml.account ='.$this->account->getId();
    }
}
```

Refactor filterReportUser function
```
protected function filterReportUser()
{
    // FILTER USER(s)
    if ($this->highest_user_level == STANDARD) {
        $conditions[] .= 'ml.user ='.$this->user->getId();
    } elseif (isset($postData['users']) && !empty($postData['users'])) {
        $accountuser = $this->doctrine->em->getRepository('models\entities\Accountuser')->findOneBy(array('user' => $postData['report_user'], 'account' => $postData['accounts'][0], 'deleted' => '0'));
    }

    if ($accountuser && $accountuser->getUserlevel()->getId() == STANDARD) {
        $conditions[] .= 'ml.user ='.$postData['report_user'];
    } elseif (count($postData['users']) != 1 || $postData['users'][0] != $postData['report_user']) {
        $or = [];
        array_map(
            function ($user) use ($or) {
                $or[] = 'ml.user='.$user;
            },
            $postData['users']
        );
        $conditions[] .= '('. implode(' OR ', $or) . ')';
    }
}
```
Final code

```
public function filterReport($postData)
{
    if (!empty($postData)) {
        $this->filterReportAccount();
        $this->filterReportUser();
    }
}

protected function filterReportAccount()
{
    // FILTER ACCOUNT(s)
    if (isset($postData['accounts']) && !empty($postData['accounts'])) {
        $or = [];
        array_map(
            function ($account) use ($or) {
                $or[] = sprintf('ml.account=%s', $account);
            },
            $postData['accounts']
        );
        $conditions[] .= '('. implode(' OR ', $or) . ')';
    } elseif ($this->highest_user_level == STANDARD) {
        $conditions[] .= 'ml.account ='.$this->account->getId();
    }
}

protected function filterReportUser()
{
    // FILTER USER(s)
    if ($this->highest_user_level == STANDARD) {
        $conditions[] .= 'ml.user ='.$this->user->getId();
    } elseif (isset($postData['users']) && !empty($postData['users'])) {
        $accountuser = $this->doctrine->em->getRepository('models\entities\Accountuser')->findOneBy(array('user' => $postData['report_user'], 'account' => $postData['accounts'][0], 'deleted' => '0'));
    }

    if ($accountuser && $accountuser->getUserlevel()->getId() == STANDARD) {
        $conditions[] .= 'ml.user ='.$postData['report_user'];
    } elseif (count($postData['users']) != 1 || $postData['users'][0] != $postData['report_user']) {
        $or = [];
        array_map(
            function ($user) use ($or) {
                $or[] = 'ml.user='.$user;
            },
            $postData['users']
        );
        $conditions[] .= '('. implode(' OR ', $or) . ')';
    }
}
```

# 3. Wrap primitives

Original code

```
public function getLogs(
    $filter = ' 1=1 ',
    $offset = 0,
    $limit = 5,
    $sortBy = 'datetime',
    $sortOrder = 'desc'
)
{
  // retrieve logs
}

// get logs
$logs = getLogs(null, 0, 5, 'datetime', 'asc');
```
If we see the above getLogs function it will be really hard to guess what each parameters passed to the function represent.
We can wrap the parameters into a class to make the function clearer.

```
class Pagination
{
  protected $filter;
  protected $offset;
  protected $limit;
  protected $sortBy;
  protected $sortOrder;
  
 // getter and setters
 ...
}

public function getLogs(Pagination $pagination)
{
  // retrieve logs
}

$pagination = new Pagination();
$pagination->setOffset(0);
$pagination->setLimit(5);
$pagination->setSortBy('datetime');
$pagination->setSortOrder('asc');
$logs = getLogs($pagination);
```
Although this approach requires more code but it makes calls to the getLogs function much easier to understand and less prone to errors.


# 4. Refactor switch statement
We have a switch statement that assign id and name variable based on the report by type.
```
// fetch result from db
$result = $query->row_array();
switch ($reportByType) {
    case REPORT_BY_USER:
        $id     = $result['userid'];
        $name   = $result['firstname'] . ' ' . $result['surname'];
    break;
    case REPORT_BY_CAMPAIGN:
        $id     = $result['campaign_id'];
        $name   = $result['campaign_name'];
    break;
    case REPORT_BY_ACCOUNT:
    default:
        $id     = $result['accountid'];
        $name   = $result['name'];
    break;
}
```
This looks ok because there are only 3 types of report by attributes. Now every time if we want to add another report by attributes we will add a new case to the switch statement, this will get messy pretty quickly.

Refactor using strategy pattern
ReportInfo class will store the information we needed from the result

```
class ReportInfo
{
    protected $id;
    protected $name;
}
```
Interface which every report type will implement
```
interface ReportByInterface {
    public function getId($reportData);
    public function getName($reportData);
}
```
Report by user type
```
class ReportByUser implements ReportByInterface
{
    public function getId($reportData)
    {
        return $reportData['userid'];
    }
    
    public function getName($reportData)
    {
        return $reportData['firstname'] . ' ' . $reportData['surname'];
    }
}
```
Report by campaign type
```
class ReportByCampaign implements ReportByInterface
{
    public function getId($reportData)
    {
        return $reportData['campaign_id'];
    }
    
    public function getName($reportData)
    {
        return $reportData['campaign_name'];
    }
}
```
Report by account type
```
class ReportByAccount implements ReportByInterface
{
    public function getId($reportData)
    {
        return $reportData['accountid'];
    }
    
    public function getName($reportData)
    {
        return $reportData['name'];
    }
}
```
Report by factory
```
class ReportBy
{
    protected $report;
    
    public function __construct($reportById) {
        switch ($reportById) {
            case REPORT_BY_USER: 
                $this->report = new ReportByUser();
            break;
            case REPORT_BY_CAMPAIGN: 
                $this->report = new ReportByCampaign();
            break;
            case REPORT_BY_ACCOUNT: 
                $this->report = new ReportByAccount();
            break;
        }
    }
    
    public function getId($reportData)
    {
        return $this->report->getId($reportData);
    }
    
    public function getName($reportData)
    {
        return $this->report->getName($reportData);
    }
}
```
We moved the switch statement to the dactory class so original code will be clean and easy to read.
```
// fetch result from db
$result = $query->row_array();
$report = new ReportBy($reportByType);
$id     = $report->getId($result);
$result = $report->getName($result);
```
Every time we want to add a new report by type we will add a new ReportByType class which implements ReportByInterface then we add the new report type to the switch statement inside the factory class so the original code stays clean.
