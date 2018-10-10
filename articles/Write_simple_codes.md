#1. Make use of conditional operators

##Elvis operator

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



#2. Low level of indentation

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

#3. Wrap primitives

```
public function get_logs(
    $filter = ' 1=1 ',
    $offset = 0,
    $limit = 5,
    $sort_by = 'datetime',
    $sort_order = 'desc'
)
```
